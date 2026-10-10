# Collection-Shaped Mixin Surfaces

Mixins frequently own a collection of related items — a `Container`'s
inventory, a `Zone`'s locations, a `HasInteractive`'s connections, an
`Exitable`'s exits. The inter-stuff contract (see
CLAUDE.md § Inter-Stuff Contract) says external
callers reach those collections through methods, not field access.
This doc says **which** methods.

The goal: given a Stuff `obj` with a collection X, an external caller
should know the surface without reading the source. Three shapes
cover almost everything; pick the one that fits the underlying
storage and stick to its canonical surface.

## Shape A — Set of items (membership-only)

The collection holds distinct items; order doesn't matter; the
operation of interest is "is X in this collection?" Backed by a
`Set<X>` internally.

**Examples:** `Zone.locations`, `HasInteractive.interactives`,
`Door.attachedTo`, `Perceptible.keywords` (semantically a set of
strings).

**Canonical surface:**

```typescript
addX(x: X): void;
removeX(x: X): boolean;        // true iff was present
hasX(x: X): boolean;
getXs(): ReadonlySet<X>;       // never raw Set
```

**Optional, only when a real lifecycle reason demands them:**

```typescript
clearXs(): void;               // destruct/reset paths only
setXs(items: X[]): void;       // admin / bulk-edit paths only
```

`clearXs` is a destructive bulk operation — adding it as a default
invites callers to use it in place of remove-loops, which silently
skips per-item invariants. Add it only when a single sweep of the
collection is the legitimate operation (e.g., `clearInteractives()`
during Avatar destruct).

`setXs` (bulk replace) has the same hazard — it bypasses per-item
add/remove notifications, and observers don't fire. Add only when a
caller has a list and the cost of computing the diff is real.

## Shape B — Map keyed by something (lookup + iteration)

The collection holds values addressable by a key independent of the
value. Backed by a `Map<K, V>` internally.

**Examples:** `Exitable.exits` (by direction string),
`CartesianZone.grid` (by `"x,y,z"` key),
`SphericalZone.focusIndex` (by rounded focus tuple).

**Canonical surface:**

```typescript
setX(key: K, value: V): void;             // or addX(value) when key derives
getX(key: K): V | undefined;              // single lookup
removeX(key: K): boolean;
hasX(key: K): boolean;
getXs(): ReadonlyMap<K, V>;               // never raw Map
```

When the key is derivable from the value (e.g., `Exitable.addExit(exit)`
where the key is `exit.getDirection()`), use `addX(value)` instead of
`setX(key, value)`. Reserve the keyed-shape `setX(key, value)` for
when key and value are independent (e.g., a property store where the
caller chooses both).

## Shape C — Ordered list (position is meaningful externally)

The collection has natural iteration order or position-based
semantics. May be backed by an array, an ordered Map, or a Set
(JavaScript Set preserves insertion order, but treating the collection
as a Set advertises that order is not load-bearing).

**Examples:** `Container.inventory` (callers expect inventory order
in display), `Named.alternateNames` (the array order is part of the
contract; first-match-wins for `removeAlternateName`).

**Canonical surface:**

```typescript
addX(x: X): void;                         // append
removeX(x: X): boolean;                   // first-match-wins removal
hasX(x: X): boolean;
getXs(): ReadonlyArray<X>;                // or a fresh `X[]` copy
```

**Optional, justified by a real use case:**

```typescript
insertX(x: X, position: number): void;    // positional ops
setXs(items: X[]): void;
```

`getXs()` may return `ReadonlyArray<X>` (the live view) or `X[]` (a
fresh copy). Prefer `ReadonlyArray<X>` for the perf reason; return
`X[]` only when the underlying storage isn't already array-shaped
(`Named.getAlternateNames` returns a copy because it filters).

## Shape D — Property bag (keyed value-only)

A specialization of Shape B where the key namespace is enumerated
(typed key objects), the read returns a typed value, and the absence
of a value is meaningful.

**Examples:** `Propertied.props` (keyed by `Property<T>`).

**Canonical surface:**

```typescript
setProp<T>(key: Property<T>, value: T): boolean;   // boolean: was a write actually accepted?
getProp<T>(key: Property<T>): T | null;            // null when absent
hasProp<T>(key: Property<T>): boolean;
removeProp<T>(key: Property<T>): boolean;
getProps(): Readonly<Record<string, PropValue>>;   // dump, for diagnostics
```

This is mostly a naming variant of Shape B. The `setProp` returning
`boolean` is the security gate's signal that the write was accepted
(see [properties.md](./properties.md)).

## Capability-derived items: the `getContents` exception

The `getXs()` rule (method named for the item type) works when the
item type describes the *relationship* between owner and item:

- `Avatar.getInteractives()` — Interactive is what they are AND what
  they are *to* the Avatar (its connections).
- `Zone.getLocations()` — same.
- `Door.getAttachedExits()` — same.

Some mixins identify their items by **capability** rather than
**kind** — `Containable`, `Sealable`, `Visible`. These are mixin
traits, not noun-phrases describing the relationship. `Container`
holds *contents*, not *containables*: to a container, its items
aren't "things-that-can-be-contained," they're its contents. Use
the relationship noun, not the capability suffix:

```typescript
// Container (Shape C, items are Containable):
addContainable(item): void;       // mutator names the capability
removeContainable(item): boolean;
hasContainable(item): boolean;
getContents(): (Stuff & Containable)[];   // accessor names the relationship
```

The mutator verbs still mention the capability (`addContainable`)
because the operation is "make this Containable belong to me" —
capability is exactly what's being asserted at the call site. The
accessor reads "give me what's *in* you," and "in" needs a
relationship noun.

**Why not just always use the capability suffix?** Future-proofing.
If a hypothetical `MassMoveOperation` mixin holds `Containable`s for
unrelated reasons, it should describe its own relationship
(`getStagedItems()`, `getTargets()`, …) instead of also being called
`getContainables()`. Two classes with the same generic accessor name
would collide semantically; classes named after distinct
relationships stay distinguishable. The same applies to any other
capability-shaped collection that comes later.

## Gameplay-named mutator pairs: the `Spawner` exception

`SpawnerMixin` exposes its collection with `trackSpawn` /
`untrackSpawn` instead of the canonical `addSpawned` /
`removeSpawned`. The "Spawner" gameplay pattern (Unity, Unreal,
Minecraft, etc.) means specifically "creates and tracks progeny";
the verb pair reads better at game-author call sites than a generic
`add` / `remove`, and the established naming carries gameplay
intent that a generic verb pair would erase.

This is an intentional one-off exception, not a license to invent
gameplay verbs for any collection. Add an exception only when:

1. The mutator action carries domain meaning beyond add/remove
   (track vs. untrack, recruit vs. discharge, enlist vs. release)
   AND
2. The domain term is established in published gameplay literature
   or a sibling engine's API surface.

If you're tempted to coin a verb pair because it "sounds nice," fall
back to the canonical surface. The collision-with-future-mixins risk
called out above still applies — gameplay verbs collide harder than
generic ones.

## The model / parlance / prose layer split

`Container.contents` is the model noun. Player-facing language
sometimes uses different words for the same concept depending on the
container's role — most prominently, MUD-tradition `inventory` for an
avatar's carried items. Don't push that vocabulary down into the
mixin or the API. The split is:

| Layer | Word | Why |
|---|---|---|
| Model / API method / internal field / MQL token | `contents` | Universal, matches the relationship, type-agnostic |
| Player command verb (the action of displaying self-contents) | `inventory` (`i`, `inv`) | MUD-canonical for the *self-look-at-what-I-carry* action |
| Narration / prose | per genre | "There is a sword here." / "Bob is here." / "The ship carries 3 passengers." Per-template, not unified |

So:

- `ContainerMixin.contents` (host-internal field)
- `Container.getContents()` (API)
- `me:contents` / `here:contents` (MQL canonical token, when the
  contents-token is added)
- `inventory` command (player verb) — reads `avatar.getContents()`
  under the hood
- `sense.survey` topic (player-action topic) — keeps
  the player vocabulary at the messaging layer where players see it

MQL aliases are fine where player vocabulary is more familiar than
the canonical noun: `me:inventory` aliasing `me:contents` is
defensible. Keep the canonical token (`:contents`) as the
documented surface so the model stays consistent; let aliases serve
ergonomics.

## Naming axes — when to pick the alternative

| Axis | Default | Use the alternative when |
|---|---|---|
| Mutator verb | `add` / `remove` | The relationship is bidirectional and the verb captures it: `attach` / `detach` for things that wire backreferences (e.g., `Door.attachExit` also updates the exit's `door` slot). Document the verb-pair locally so future readers don't assume add/remove. |
| Membership predicate | `hasX(item)` | Don't deviate. `contains(item)` is grammatical but breaks grep across mixins. |
| Add return | `void` | `boolean` only when validation can reject the add (e.g., `Exitable.addExit` returns false for invalid direction). |
| Remove return | `boolean` (was-it-there) | `void` only for `clearXs`-style bulk drops. |
| Bulk replace setter | (omit) | Real admin / import case; opt in deliberately. |
| Clear | (omit) | Destruct cleanup or explicit reset only. |

## Internal storage choices

The doc above is about the **external** surface. Internally, mixins
choose storage to suit their access patterns:

- **Hot membership tests** (`hasX` called per-frame) → `Set<X>`.
- **Order-preserving with O(1) end-append** → `X[]` or `Set<X>`
  (Set's insertion-order iteration is reliable in modern JS).
- **Keyed lookup** → `Map<K, V>`.
- **Tree shapes** → `Map<K, Detail>` with nested `details: Map<...>`
  (Detailed's pattern).

Storage is host-internal; switching it is a non-breaking change
provided the public surface keeps the canonical shape.

## Why not just expose the field?

Because:

1. The methods-only contract lets shadows, audits, and decorators
   intercept reads and writes uniformly. Field access bypasses
   everything.
2. Returning the raw collection invites callers to mutate it. A
   `ReadonlyMap` / `ReadonlySet` is a typed signal, but only TypeScript
   enforces it; the JS-level guarantee is that the only writers are
   inside the host class body.
3. Surfaces drift. Today `inventory` is a Set; tomorrow it's a
   priority queue. With methods-only, that's a private refactor;
   with public fields, every caller breaks.

## Cross-references

- CLAUDE.md § Inter-Stuff Contract — the broader
  rule the shapes here implement.
- [mixins.md](./mixins.md) — how mixins are composed and detected.
- [properties.md](./properties.md) — Shape D in detail (Propertied).
- [antipatterns.md](../antipatterns.md) — what NOT to do at the call
  site (`obj.field` reads, raw `Map` returns, etc.).
