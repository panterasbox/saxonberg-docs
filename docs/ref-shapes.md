# Ref shapes — how Stuff references other Stuff

Saxonberg has one way a `Stuff` field references another `Stuff`, and
it has **two axes**. This doc gives the model, the method-surface
conventions, and the antipatterns.

## The two axes

**Axis 1 — what are you pointing at?** This is a question about *your
own meaning*, not about the target:

| you mean | you store | persistable? |
|---|---|---|
| **identity** — "what kind of thing" (`my species is human`) | a `templatePath` string | **yes** |
| **instance** — "this particular object" (`the room I am in`) | a **live ref** | **never** |

The discriminator is **not** a property of the target. A room has a
perfectly good unique templatePath, and you still hold it as an
*instance* when you mean "the one I'm standing in". A species would
still be an *identity* ref if somehow two existed. Only the author
knows which question the field is asking — which is exactly why it is
**declared**, not inferred (see `fieldMeta` below).

**Axis 2 — what happens when it dies?** Applies to **instance refs
only**; an identity ref re-resolves from its path, so it cannot dangle
and needs no rule.

| lifetime | meaning |
|---|---|
| `weak` | the getter self-heals — null the slot when the target is destroyed |
| `symmetric` | paired both ways; each side clears the other's back-ref on destruct |
| `owned` | the holder's lifetime bounds the target's; destruct cascades |

### The foundational constraint

**`stuffId` does not survive a reboot.** Everything above follows from
it:

- An identity ref persists, because a `templatePath` is stable.
- An instance ref **cannot** persist. Not by convention — there is
  nothing durable to write down. Persisting an instance *relationship*
  is done **structurally** (containment capture / a `props`
  manifest rebuilds it at hydrate), never by storing a reference.
- Declaring a field both `instance` and `persistent` is therefore a
  build-time error, not a style violation.

### Live ref, not stuffId

For an instance ref, **store the live ref, not the `stuffId` string.**
They name the same thing; the live ref is the one that has already
paid for the lookup. A stuffId handle is what you reach for when the
lifetime mechanism is missing — it is stringly-typed, loses narrowing,
and is weak only by convention (nothing stops a caller caching the
resolved object anyway).

### Declaring it

Both axes live in one field-keyed declaration:

```ts
static fieldMeta = {
  _speciesPath: { ref: 'identity', persistent: true },
  _container:   { ref: 'instance', lifetime: 'weak' },
  _hauling:     { ref: 'instance', lifetime: 'symmetric', inverse: '_hauledBy' },
  exits:        { ref: 'instance', lifetime: 'owned' },
};
```

The invalid combinations are checked **at class registration**, and
throw naming the field and the class:

- `instance` with `persistent` — nothing durable to write down;
- `identity` with any `lifetime` — it re-resolves and cannot dangle;
- a `lifetime` with no `ref: 'instance'`;
- `symmetric` with no `inverse`;
- an `inverse` this class declares that does not point back.

Validation runs on the **merged** metadata, so a base declaring
`ref: 'instance'` and a subclass sharpening it with a `lifetime` is
legal and is judged as one entry. A *cross-class* pair
(`Adornment.adornedTo` ↔ `Adornable.fixtureSlots`) lives on two hosts
and registration of one cannot see the other — that reciprocity belongs
to `pnpm lint:field-meta`, which sees the whole tree statically. The
same lint carries the rules that are wrong within a single class body
whatever other layers say: an unknown entry property, a malformed
value, and a legacy static coming back.

> **Historical note.** This doc previously described *three* patterns:
> A (string-stored singleton), B (live ref), and C (path-resolved
> cross-scope singleton). A and C stored the **identical thing** — a
> templatePath string — and differed only in what the getter did with
> it (compare vs resolve), which is a usage decision, not a
> representation. They are one thing: an **identity** ref. B is an
> **instance** ref. The letters have been swept out of the docs and the
> source comments; they survive only in the section headers below (as
> `was: Pattern X` signposts) and in the R2.x rule labels, which are
> still how the cleanup rules are referred to across the tree.
>
> Unrelated collision: `slot.md` has its own Pattern A/B/C — how a host
> *provides* slots. That taxonomy is untouched.

---

## Known gaps — sites not yet declared

The reference-lifetime build declared 16 fields and left a tail. None of
these is a live bug: every one already guards on `isDestroyed()` at its
readers, which is why they sorted to the bottom of the sweep. They are
recorded here rather than in a plan doc because the plan is retired and
this file is the live reference.

**Instance refs still relying on a hand-written guard** (each wants a
`weak` declaration and its guard deleted):

- `SandboxCrossingExit.crossing`
- `ExitableVessel.outCache` / `entryCache` — note these pair a live
  `Exit` with a **stuffId invalidation stamp** (`outCacheEnvId`), which
  is not a reference: it is compared for equality and never resolved.
- `LoungeWarren._reapTimers`
- `DormWarren._unitsByKey` / `_corridorsByFloor` / `_doorsByKey`

**Held-side R2.4 unhooks not folded into declarations** — `Containable`,
`Spawned`, `WarrenMember`, `AetherHosted`, `Slottable`, `Slotted`. All
six are `cleanupOnDestruct`-form, so each must convert **atomically**
(slot 3 runs after slot 2.5, so a declaration left alongside its handler
would have 2.5 destruct first and the handler then walk destroyed
objects). Each also needs its unhook separated from surrounding policy:
`Containable`'s goes through `ContainmentApi.move`, the chokepoint, and
`Slotted`/`Slottable`'s `onSlotReleased` notification is policy by the
`owned` audit's own step 4.

**`ref: 'identity'` is declared nowhere yet**, and that is expected
rather than an omission: the framework attaches no *runtime* behaviour to
the identity axis — a path resolved on read cannot dangle into freed
memory, so there is nothing for the proxy or the destruct slot to
enforce. The declaration is documentation-only there today. It IS
validated (`identity` + any `lifetime` throws), so declaring one is safe
whenever the documentation value is wanted.

⚠ **It does not follow that an identity ref cannot be wrong.** A path
naming no row resolves to `null` forever, and the failure is *silent* —
the TPA reform's `mainsRef` is the exemplar: a gate wired to a
mistyped line reports no supply, for a reason no author could find by
looking at the gate. **The enforcement is build-time, not runtime:
`pnpm lint:census` walks every template-path-valued field in every
shipped row and fails on one that resolves to nothing.** ⚠ Its field
list (`refsOf` in `scripts/check-template-census.ts`) is **enumerated by
hand**, so a new identity-ref field is not covered until it is added
there — one line, and the gate is the only thing standing between a typo
and a device that quietly does nothing.

---

## Identity refs (was: Pattern A — string-stored singleton ref)

**Stores**: a `string` (templatePath or short name) identifying a
singleton Idea.

**Use when**:

- The target is a **singleton** (one canonical instance per
  templatePath: `Material`, `Species`, `BodyPlan`, `LocomotionMode`,
  `Clade`, etc.).
- Comparison is by template identity, not instance identity ("is this
  actor's species `Homo sapiens`?", not "is this *the same instance*
  of the Homo sapiens singleton?").
- The field is runtime-only OR persisted (an identity ref handles both
  cleanly).

**Why this shape**: tiny memory; trivial JSON/Mongo round-trip; the
identifier survives hot-reload of the singleton (path stays valid
even as the live singleton churns); no marshaller ceremony for
runtime fields.

### Field naming

```ts
protected _xxxPath: string | null = null;
```

- **Private** with `_` prefix — the field is host-internal; external
  reads/writes go through methods.
- **`Path` suffix** on the field name — honest about what's stored
  (an identifier, not a singleton).
- **`null`-default** when the reference is optional.

The `_` prefix + private modifier is consistent with the project's
"inter-Stuff contract: methods only" rule (CLAUDE.md). Hydrator
bracket-assigns onto the private field by name; external code never
touches the underscore.

#### Public-surface naming uses the bare conceptual name

The `Path` suffix discussed above is for the **backing field only**.
The **public surface** — the entry in `persistentFields`, the YAML
key authors write, the `getXxx` / `setXxx` method names — uses the
**bare conceptual name** without a `Path` / `TemplatePath` suffix.

Examples in current substrate: `container`, `props`, `cast`, `destination`,
`door`, `attachedHosts`. Not `containerPath`, not `attachedHostPaths`.

Reasoning: the identity-ref type signature (`string`) plus the field's
documentation as a singleton ref already convey "stored as a path";
the `Path` suffix on the public surface asks readers to re-confirm
what the type already shows. The bare conceptual name reads more
naturally in YAML and matches the established convention across the
substrate.

The backing field MAY drop the `Path` suffix for parallel uniformity
(`protected _attachedHosts: [string, string] | null`) — internal
choice, since callers never see backing-field names.

### Method surface

```ts
interface Xxxed {
  /** Convenience getter — resolves to the singleton (null if not loaded). */
  getXxx(): Xxx | null;

  /** Setter — accepts the singleton (or null); stores its path internally. */
  setXxx(value: Xxx | null): void;

  /** Optional predicate — accepts either form for ergonomics. */
  isXxx(value: Xxx | string): boolean;
}
```

**Add a raw-path getter only when a real consumer needs it.** The
default surface is singleton-form only — callers who want the live
singleton get it; nobody else has to think about the path. Add
`getXxxPath(): string | null` only when some consumer genuinely needs
to key off the path without resolving (e.g.,
`Species.getBodyPlanPath()` exists because `Wearable.slotClaims`
keys off the body-plan path string for per-body-plan vocabulary).

**No raw-form setter** in the default surface. The singleton-form
setter is the public write path; tests / hydration / framework code
that genuinely need to write a string without a singleton handle
go through bracket-assign on the private field directly (Hydrator
already does this).

### Setter behavior

```ts
public setXxx(value: Xxx | null): void {
  this._xxxPath = value === null ? null : value.getTemplatePath();
}
```

Singleton-in → path-out; null is always allowed. No further
validation — the substrate doesn't gate against unloaded singletons
because:

- The singleton may not exist yet at construction time.
- Hot-reload may temporarily disrupt the lookup.
- The Api-side resolver returns `null` for unknown paths; that's the
  graceful failure path.

### Getter behavior

```ts
public getXxx(): Xxx | null {
  if (this._xxxPath === null) return null;
  return XxxApi.findByPath(this._xxxPath) ?? null;
}
```

`null` field → `null` return. Non-null field + unresolvable singleton
→ also `null` (Api-side lookup returns null for unloaded). Same
semantics across all singleton refs.

### Existing exemplars

| Site | Field | Methods |
|---|---|---|
| `Tangible` | `_materialPath` (+ per-Detail `_detailMaterialPaths` map) | `getMaterial(detailKey?)` / `setMaterial(value, detailKey?)` |
| `Organism` | `_speciesPath` | `getSpecies()` / `setSpecies(value)` |
| `Species` | `_bodyPlanPath` | `getBodyPlan()` / `setBodyPlan(value)` / `getBodyPlanPath()` (raw — consumed by slot claims) |
| `Species` | `_parentCladePath` | `getParentClade()` / `setParentClade(value)` |
| `Species` | `_defaultMaterialPath` | `getDefaultMaterial()` / `setDefaultMaterial(value)` |
| `Mobile` | `_engagedModePath` (runtime-only) | `getEngagedMode()` / `setEngagedMode(value)` / `isEngagedIn(value)` |
| `Drivable` | `_vehicularModePath` | `getVehicularMode()` / `setVehicularMode(value)` |
| `BodyPlan` | `defaultLocomotionMode` (short-name variant; no `_xxxPath`, no leading slash) | `getDefaultLocomotionMode()` / `setDefaultLocomotionMode(value)` |
| `Atmospheric` | `_biomePath` | `getBiome()` / `setBiome(value)` ([biome.md](./subsystems/biome.md)) |
| `Biome` | `_extendsBiomePath` | `getExtendsBiome()` / `setExtendsBiome(value)` / `getExtendsBiomePath()` (raw — consumed by `BiomeApi`'s ancestry walker) |
| `Party` | `formationPath` (raw-path variant — the holder **never resolves**: the party side stores/returns the string only and never imports `lib/combat`; the consumer (combat) resolves path → `CombatFormation` Idea on its own side of the one-way dep, via the total `PartyApi.formationPathOf` chain) | `getFormationPath()` / `setFormationPath(value)` |
| `Character` | `_domicileAddress` (ADDRESS-namespace path, not templatePath — the raw-path variant: the holder never resolves; `GovernmentApi.residentOf` resolves address → jurisdiction chain on the civics side; setter enforces persists-until-replaced) | `getDomicileAddress()` / `setDomicileAddress(value)` ([civics.md](./subsystems/civics.md)) |
| `Locality` | `_governmentKey` (durable-`key` join variant, the `_brandKey` shape — resolves on read through `GovernmentCatalogue`) | `getGovernmentKey()` / `setGovernmentKey(value)` ([civics.md](./subsystems/civics.md)) |
| `Seed` | `growsIntoPath` (**template**-path variant — the holder never resolves it to an Idea; the `plant` verb hands it to `StuffApi.clone` to mint a fresh instance, so the string names a *lineage to instantiate*, not a singleton to read) | `getGrowsIntoPath()` / `setGrowsIntoPath(value)` ([husbandry.md](./subsystems/husbandry.md)) |
| `Plant` | `seedTemplatePath` (the same instantiate-don't-resolve variant, in reverse — a flowering episode clones it into the pot) | `getSeedTemplatePath()` / `setSeedTemplatePath(value)` |

---

## Instance refs (was: Pattern B — live Stuff reference)

**Stores**: a direct reference to the live target Stuff object.

**Use when**:

- The target is a **specific instance** (a chair, an NPC, an item)
  whose identity-as-instance matters.
- The consumer reads target properties **frequently** in hot paths
  (containment walks, slot occupancy checks, perception scans).
- The field is **runtime-only**. Live refs are transient by
  definition — see the foundational constraint at the top of this
  doc; persisted "live ref" fields don't exist in the substrate.

**Why this shape**: direct property access without Api lookup;
TypeScript types catch mistakes; substrate invariants enforce
cleanup through the dispatcher.

### Field naming

```ts
protected _xxx: Stuff & XxxType;
// or for collections
protected _xxxs: Set<Stuff & XxxType>;
protected _xxxByKey: Map<string, Stuff & XxxType>;
```

- **Private** with `_` prefix.
- **No suffix** — the field stores the live ref, no need to disambiguate.

### Method surface

```ts
interface XxxOwner {
  getXxx(): Stuff & XxxType;     // or | null if optional
  setXxx(value: Stuff & XxxType): void;
}

// For collections (per docs/subsystems/collections.md):
interface XxxsOwner {
  getXxxs(): ReadonlySet<Stuff & XxxType>;
  addXxx(value: Stuff & XxxType): void;
  removeXxx(value: Stuff & XxxType): void;
  hasXxx(value: Stuff & XxxType): boolean;
}
```

Same getter/setter naming as an identity ref — the difference is the
RETURN type, not the method names. A caller writing `obj.getMaterial()`
doesn't know (or care) whether it's resolved from a stored path or
returned from a live ref; the contract is the same.

### Persistence (or lack thereof)

Live refs are **transient by definition**. They reset on clone /
hydrate to whatever default the field declares (null for a single
ref; empty collection for Set/Map). The substrate doesn't ship a
marshaller-for-live-refs path because no field in `lib/` needs one
— every container-shaped relationship rebuilds at hydrate via
template seeding (and per-character session state, for inventory).

If a relationship genuinely needs to survive saves, it lives one
layer up: either (a) the field stores a path string (an identity
ref, resolved eagerly or on read), or (b) a higher-layer mixin owns
the persistent shape and
hydrates the live refs from that.

### Instance-ref sub-flavors (cleanup story)

Within instance refs there are four structural sub-flavors,
distinguished by the cleanup rule that applies when one of the two
sides destructs. These are the **R2.1–R2.4 cleanup rules**:

> **Residency corollary.** The same attachment relationships that need
> coordinated cleanup on destruct need *veto-coupling on evict*: an
> object in an R2.1-owned or R2.2-symmetric relationship vetoes
> `canEvict` while its anchor is alive (a shadow while its host lives, an
> exit while its room lives, contents while their container is non-empty).
> The self-eviction sweep derives its relational veto roster from these
> rules — see [residency.md](./subsystems/residency.md).

#### R2.1 — Owning cascade

When the held side's lifetime is bounded by the holder's (the
held thing has no independent existence outside the holder), the
holder's `onDestruct` destructs each owned thing before chaining
`super.onDestruct()`.

- **Mechanism**: eager, convention-based.
- **Enforcement**: convention. Failure-mode is "owned objects
  leak" — caught later by GC; doesn't corrupt invariants.
- **Exemplars**: `Exitable.onDestruct` (outbound Exits),
  `Adornable.onDestruct` (fixtures). `AetherMixin.cleanupOnDestruct`
  destructs every hosted update (`AetherHosted` Ideas) — the
  must-be-hosted invariant gives updates no independent existence, so
  this owning cascade is framework-enforced (`cleanupOnDestruct`)
  rather than convention.

#### R2.2 — Symmetric two-way pair

For paired bidirectional refs (both sides hold each other), each
side's setter atomically updates both sides; each side's
`onDestruct` clears the back-ref on the other.

**Declare it**, naming the field on the other side:

```ts
static fieldMeta: FieldMeta = {
  _hauling: { ref: 'instance', lifetime: 'symmetric', inverse: '_hauledBy' },
};
```

- **Mechanism**: eager via setter; eager on destruct, in slot 2.5.
- **Enforcement**: framework. `symmetric` with no `inverse` throws at
  registration, and a same-class `inverse` that does not point back
  throws too. A *cross-class* pair is the whole-tree lint's job —
  registering one host cannot see the other.
- **All symmetric clears run before all owned cascades** within 2.5, so
  a host that is both (see `Boundary`) clears back-refs while its owned
  targets are still alive.
- **Exemplars**: `Hauler` ↔ `Haulable` (the hitched-cart coupling —
  `hauler._hauling` ↔ `cart._hauledBy`, runtime-only; see
  [conveyance.md § Haulage](./subsystems/conveyance.md#haulage--pulling-a-cart)),
  `Exit.inverse` (which was half-symmetric until this was declared).

> **Two sites this rule does NOT cover, and why.**
>
> **`Adornment.adornedTo` ↔ `Adornable.fixtureSlots` is not a symmetric
> pair**, though this doc asserted it was — and asserted it while the
> Adornment half did not exist in code at all. `fixtureSlots` is
> `owned`, a field carries exactly one lifetime, and the holder
> destructs its fixtures anyway. The back-ref only needs to stop
> dangling when a fixture dies standalone, which is `weak`.
>
> **`BoundaryAnchor.boundary` is not expressible.** The reciprocal slot
> is `anchorA` *or* `anchorB` depending on which side the anchor sits,
> and `inverse` names exactly one field; `Boundary._clearAnchor` is
> side-agnostic by design. It stays hand-written rather than acquiring a
> wrong declaration.

#### R2.3 — Read-side self-heal (declared)

**You no longer write this.** Declare the field and the framework does
it:

```ts
static fieldMeta: FieldMeta = {
  _x: { ref: 'instance', lifetime: 'weak' },
};

public getX(): (Stuff & XxxType) | null {
  return this._x;      // that's the whole getter
}
```

Every `ref: 'instance'` field self-heals on read, whatever its
`lifetime` — axis 2 names only the *destruct-side* rule, so `weak`
means "no destruct-side rule", not "the only one that heals".

- **Mechanism**: the `ProxyApi` get trap. The weak-field set is hoisted
  into the handler closure once per `wrap()`, so the cost on a class
  with no instance refs is a single `!== null`.
- **Arity is decided at heal time, not declared.** The trap heals a
  slot holding a lone Stuff that answers `isDestroyed()`, and leaves
  `Array` / `Set` / `Map` alone — walking a collection on every read is
  O(n) on hot paths and buys nothing the R2.4 chokepoints don't. A
  collection gets a destruct-side rule instead.
- **⚠ Reads on the RAW target do not heal.** The trap is the mechanism,
  so `Stuff.RAW_TARGET` / `ProxyApi.unwrap` and the deliberately-raw
  residency sweeps see the un-healed slot. A site that chooses raw
  carries its own `isDestroyed()` guard — `ResidencyLogic` is the
  worked example.
- **Exemplars**: `Containable.getContainer()` (the backstop for a
  Container destructed without first evacuating — S1/S2),
  **`Containable.getRestingOn()`** (a destroyed supporting surface),
  `Spawned.getSpawner()`, `Hauler.getHauledCart()`,
  `Haulable.getHauledBy()`, `WarrenMember.getWarren()`. All six were
  hand-written; all six are now one-line reads.

#### R2.4 — Collection symmetric cleanup (framework-enforced)

For any instance-ref collection of live refs (Set/Map of `Stuff & X`),
the held side MUST register a framework `static cleanupOnDestruct`
on its mixin that unhooks itself from every collection it's a
member of, via the canonical mutation chokepoint
(`ContainmentApi.move`, `Slotted.vacate`, `Spawner.untrackSpawn`,
etc.).

- **Mechanism**: eager, framework-enforced.
- **Enforcement**: framework. `StuffApi.destruct` dispatches via
  the mixin chain; subclass `onDestruct` overrides cannot bypass.
- **Walk order**: most-derived first / base last. For a host
  composing `ContainerMixin(ContainableMixin(Stuff))`, Container
  fires first (evacuates contents while `_container` is still
  set), then Containable (unhooks the destructed item from any
  outer container).
- **Exemplars**:
  - `Containable.cleanupOnDestruct` → `Container.contents`
  - `Container.cleanupOnDestruct` → evacuate contents to outer
  - `Slottable.cleanupOnDestruct` → `Slotted.slots` on every host
  - `Slotted.cleanupOnDestruct` → active vacate of every occupant
  - `Spawned.cleanupOnDestruct` → `Spawner._spawned`
  - `AetherHosted.cleanupOnDestruct` → host's `_hostedUpdates` (via
    the `_dropHostedUpdate` chokepoint, when destructed standalone
    while still hosted)
  - `Species.onDestruct` (concrete-class form) → `Clade.species`

R2.4 is the load-bearing rule of the cleanup story — it's the
reason `StuffApi.destruct` grows the mixin-registry dispatch step,
and it's the only one of the four where the framework actively
enforces correctness rather than trusting authoring discipline.

See [`docs/subsystems/mixins.md` § `cleanupOnDestruct`](./subsystems/mixins.md)
for the static-shape convention.

### Existing exemplars

| Site | Field | Cleanup rule | Notes |
|---|---|---|---|
| `Container` | `contents` | R2.4 (Container side) | Runtime-only; evacuates on destruct |
| `Containable` | `environment` | R2.3 (declared) + R2.4 (held side) | `{ ref: 'instance', lifetime: 'weak' }` + framework cleanup |
| `Containable` | `_restingOn` | R2.3 (declared) | `{ ref: 'instance', lifetime: 'weak' }`; a destroyed surface reads null |
| `Slotted` | `slots` | R2.4 (holder side) | Runtime-only; active vacate fires `onSlotReleased` |
| `Slottable` | (none — held side) | R2.4 | Static cleanup walks every host |
| `Adornable` | `fixtureSlots` | `owned` (declared) | Holder destructs each fixture |
| `Adornment` | `adornedTo` | `weak` (declared) | **Not** the symmetric pair this table used to claim — see below |
| `Exitable` | `exits` | `owned` (declared) | Holder destructs each outbound Exit; the `setBlocked` pre-pass stays policy |
| `Boundary` | `anchorA` / `anchorB` | `owned` (declared) | `detach()` deliberately NOT called on destruct — it nulls the slots |
| `BoundaryAnchor` | `boundary` | hand-written | **Not expressible**: the reciprocal slot is `anchorA` *or* `anchorB` by side, and `inverse` names one field |
| `Exit` ↔ `Door` | both sides | R2.2 (symmetric) | Convention-based reciprocal clear |
| `Exit` | `inverse` | `symmetric` (declared) | Was HALF-symmetric — cleared on its own destruct, never on its partner's |
| `Exit` | `source` / `_destination` | `weak` (declared) | |
| `DoorBearing` | `door` | `weak` (declared) | The mixin had no destruct hook at all |
| `Aether` | `_hostedUpdates` | `owned` (declared) | Converted atomically from `cleanupOnDestruct` |
| `Hauler` ↔ `Haulable` | `_hauling` / `_hauledBy` | R2.2 (symmetric) + R2.3 (declared) | `hitch`/`unhitch` atomic; `onDestruct` reciprocal clear; runtime-only |
| `Spawner` | `_spawned` | R2.4 (held side via `Spawned`) | Runtime-only; transient |
| `Spawned` | `_spawner` | R2.3 (declared) + R2.4 | `{ ref: 'instance', lifetime: 'weak' }` + static unhook |
| `WarrenMember` | `_warren` | R2.3 (declared) + R2.4 | `{ ref: 'instance', lifetime: 'weak' }` + static unhook |
| `Clade` | `species` | R2.4 (held side via `Species`) | `Species.onDestruct` chains the unhook |

---

## Identity refs, resolved on read (was: Pattern C)

**Stores**: a templatePath string. The getter resolves on every
read via `StuffApi.findByTemplatePath`. **No runtime cache slot.**

**Use when**:

- The target is a **singleton-by-convention** Stuff (one instance
  per templatePath) that may live in a different load scope
  (different zone, hot-reload churn).
- The holder shouldn't hold a live ref because cross-scope identity
  is unstable: hot-reload churns the singleton; zone loading is
  lazy.
- Construction-time cyclic resolution is a concern.

Resolve-on-read stays **singleton-only**. The "generalize it to
instances" idea is dropped under the foundational stuffId
constraint — a cross-scope ref to a multi-clone instance can't be
keyed by stuffId (doesn't persist), and a live ref doesn't survive
target unload. Within-session live refs to non-templated targets
are instance refs; cross-reboot is unsupported until full game-state
dump exists.

### Field shape

```ts
protected _xxxPath: string | null = null;
```

That's it. No `_resolved` cache slot. The resolved-cache pattern
was dropped: the `byTemplatePath` index is O(1), the cache added a
self-heal complication for no measurable gain, and a stale cache
slot was a real hot-reload hazard.

### Method surface

```ts
interface PathRefXxxed {
  getXxx(): (Stuff & XxxType) | null;   // resolves on every call
  setXxx(value: Stuff & XxxType): void;  // stamps the path
  // Optional, only when zone-faulting is needed:
  async resolveXxx(): Promise<Stuff & XxxType>;  // async load on miss
  // Optional, only when a real caller needs the raw form:
  getXxxPath(): string | null;
}
```

The async `resolveXxx()` is an **Exit-specific affordance**, not a
resolve-on-read requirement. It exists because Exit destinations may
trigger zone-load faults during `Mobile.traverse`. Resolve-on-read
fields that target already-loaded singletons should skip it.

### Existing exemplars

| Site | Field | Notes |
|---|---|---|
| `Exit` | `_destinationPath` | Cross-zone exits; resolves via `findByTemplatePath` every read; `resolveDestination()` for the zone-fault async path. No runtime cache. |

---

## Decision matrix

Two questions, in order.

**1. What am I pointing at?**

| | identity ref | instance ref |
|---|---|---|
| You mean | "what kind of thing" | "this particular object" |
| Stores | a `templatePath` string | a live ref |
| Persists? | yes | **never** (stuffId is ephemeral) |
| Survives target hot-reload replacement? | yes — re-resolves | no |
| Read cost | a lookup (cache only with a real reason) | direct |
| Construction-time cycles | handled (resolve is deferred) | may not have the ref yet |
| Comparison | path equality, honest across reload | reference equality |

If the field would be **persisted**, the answer is identity — there is
no other option, because an instance reference has nothing durable to
write down.

**2. If instance — what happens when the target dies?**

| | `weak` | `symmetric` | `owned` |
|---|---|---|---|
| Target tracks me back? | no | yes | no (I bound its life) |
| On its destruct | my getter nulls the slot on next read | it clears my back-ref eagerly | n/a |
| On my destruct | nothing | I clear its back-ref | I destruct it |
| Needs `inverse`? | no | **yes** | no |
| Failure if you get it wrong | a dangling ref (recoverable) | one stale back-ref (recoverable from the other side) | **destroys a live object you did not own** |

`owned` is the dangerous one — it is the only lifetime that *destroys*
something, so it is the one to be sure about. Note that
`Container.cleanupOnDestruct` is deliberately **not** a plain `owned`
cascade: it evacuates contents to the outer container first and
destructs only as a last resort. An `owned` declaration means "this
thing has no existence without me", not merely "I hold it".

## Antipatterns

Things to avoid when working with refs to other Stuff.

### A.1 — Naming a string-storing field after the singleton type

```ts
// WRONG: misleading name
protected material: string | null;
public getMaterial(): string | null { return this.material; }

// RIGHT: name reflects storage
protected _materialPath: string | null;
public getMaterial(): Material | null { ... }  // resolves
```

The convenience getter `getMaterial()` should return a Material,
not a string. Callers who think "I'm asking for the material" should
get the material, not have to do a second lookup.

### A.2 — Exposing only the raw form when the singleton is what callers want

```ts
// WRONG: every caller has to do the lookup themselves
public getMaterialPath(): string | null;
// (no getMaterial() convenience)
```

Hot-path consumers (perception, render, validation) end up duplicating
the lookup across the codebase. Expose the convenience getter as the
default; expose the raw getter only when a real consumer needs it.

### A.3 — Adding a raw setter "for symmetry" without a real consumer

```ts
// WRONG without a real caller
public setMaterialPath(path: string | null): void { ... }
```

The singleton-form setter is the public write path. Tests and
framework code that need to write a string without a singleton in
hand go through bracket-assign on the private field directly (per
the Hydrator pattern).

Add a raw setter only when a real caller is forced to a string
write (e.g., a YAML loader that wants to defer singleton resolution
until later in bootstrap).

### A.4 — Holding an *identity* question as an *instance* ref

```ts
// WRONG: the field means "what material is this made of" —
// an identity question — but stores a live ref.
protected _material: Material | null;
```

**This entry used to say "singletons should be referenced by path."
That framing was wrong**, and it invited a fair objection: a singleton
and a clone are both just `Stuff`, so why would the target's instance
count change how you point at it? It doesn't. What matters is which
question the *field* is asking (see [The two axes](#the-two-axes)).

`_material` means *"what kind of matter is this"* — identity. Store the
templatePath. `_container` means *"the specific room I am in"* —
instance. Store the live ref. Neither choice is read off the target.

The concrete failure when an identity question is held as an instance
ref is **replacement, not destruction**:

- Code hot-reload works by destructing a cached singleton so the next
  call **lazy-re-creates it at the same templatePath with a new
  stuffId** (see [hot-reload.md](./subsystems/hot-reload.md)).
- A live ref across that self-heals to `null` — and the relationship is
  gone **permanently**, even though a perfectly good instance now
  exists under exactly the name you meant.
- An identity ref re-resolves the path and finds it.

That is the whole hazard, and it is why `weak` cannot substitute for
getting axis 1 right: self-healing to `null` is the *correct* answer
for an instance that died, and the *wrong* answer for an identity whose
current holder was swapped.

> **Not a hazard**: `TemplateApi.restoreFromTemplate` (content go-live)
> mutates the existing instance and **preserves its stuffId**, so it
> does not strand a live ref. Only code hot-reload replaces the object.
> The older version of this entry overstated this.

**Not an instance of this antipattern**: a live ref held *genuinely
transiently* — a constructor parameter consumed during `postRegister`,
handed to a setter that stores the path, **and then released**. Judge the
**durable** field, not every variable that briefly holds an object.

> The emphasis on *released* is load-bearing, and `Shade`/`WireBody` are
> the worked example. Both take a `species` ctor argument and hand it to
> `setSpecies()` — so the durable field is `OrganismMixin._speciesPath`,
> correctly an identity ref. They used to retain the ctor's live
> `Species` for the object's whole life, which made the carve-out true
> of the *pattern* but not of those two sites; both now null the slot
> the moment `setSpecies()` returns.

### B.1 — Persisting a live ref

```ts
// WRONG: live refs are transient by definition. This now THROWS at
// class registration — `ref: 'instance'` with `persistent` is a
// build-time error, not a style violation.
static fieldMeta: FieldMeta = {
  _container: { ref: 'instance', persistent: true },
};
```

Live refs are runtime-only. Persistence needs either an identity ref
(store a path) or a higher-layer mixin that owns the persistent
shape (e.g., a templated `props` manifest that re-creates the
relationship at hydrate).

### B.2 — Using a live ref for a cross-scope singleton

```ts
// WRONG: live ref to something that lives in another load scope
protected _destination: Stuff & Container;
```

If the holder and target may load separately (different zones,
hot-reload), a live ref goes stale or holds a destroyed instance.
Use an identity ref — store the path, resolve on read.

### B.3 — Holding an asymmetric single without R2.3 self-heal

```ts
// WRONG: getter returns potentially-destroyed ref
public getCage(): (Stuff & Cage) | null {
  return this._cage;
}
```

When the holder doesn't get framework cleanup notification (the
target doesn't track who points at it), the read must self-heal —
which you get by **declaring** the field, not by writing it:

```ts
static fieldMeta: FieldMeta = {
  _cage: { ref: 'instance', lifetime: 'weak' },
};

public getCage(): (Stuff & Cage) | null {
  return this._cage;      // the trap does the rest
}
```

So the "wrong" version above is now wrong only by omission: the getter
body is right, and what is missing is the declaration.

### B.4 — Holding a collection of live refs without R2.4 symmetric cleanup

```ts
// WRONG: collection accumulates destroyed entries; iteration walks
// destroyed objects; substrate invariants drift.
class FactionMembership extends ... {
  private members: Set<Character> = new Set();
  // (no static cleanupOnDestruct — collection leaks dead refs)
}
```

Mixins that hold a collection of live Stuff refs (or that are the
held side of one) MUST register a `static cleanupOnDestruct(stuff)`
that unhooks via the canonical mutation chokepoint. The dispatcher
walks the mixin chain on every destruct; subclass overrides cannot
bypass it.

---

## Where to put new singleton refs

When introducing a new singleton-ref field:

1. Pick the field name: `_xxxPath` (private, `Path` suffix).
2. Add to `persistentFields` if the ref should survive saves.
3. Implement `getXxx(): Xxx | null` using the relevant Api's
   findByPath / findByTemplatePath / etc.
4. Implement `setXxx(value: Xxx | null)` extracting
   `value.getTemplatePath()`.
5. Add a `getXxxPath(): string | null` raw getter ONLY if a
   specific consumer needs it (and document the consumer in a
   comment).
6. Add an `isXxx(value: Xxx | string)` predicate ONLY if callers
   genuinely want the polymorphic form (most don't).
7. Hydrator handles persistence automatically — it bracket-assigns
   the path string onto `_xxxPath` directly.

When introducing a new live-ref field:

1. Pick the field name: `_xxx` (private, no suffix).
2. The field is runtime-only — do NOT add to `persistentFields`.
3. Implement `getXxx(): Xxx` and `setXxx(value: Xxx)`. For
   asymmetric singles, declare `{ ref: 'instance' }` — the R2.3
   self-heal is framework-run, not hand-written.
4. For collections, follow the patterns in
   [docs/subsystems/collections.md](./subsystems/collections.md)
   AND register `static cleanupOnDestruct` per R2.4.

When in doubt: an **identity** ref for singleton Ideas (Material /
Species / BodyPlan / Clade / LocomotionMode), an **instance** ref for
everything else — and when a singleton needs cross-scope
addressability, an identity ref that resolves on read.

---

## Identity, lineage, and backing (the antipattern-sweep doctrine)

Three facts about a Stuff that the codebase historically let one field
blur together, now named separately:

- **Lineage is the class.** What kind of behavior this is, in code —
  the thing `FromModule`, the top-level-branch check, and hot-reload
  identity already trust. Lineage never lives in data.
- **`templatePath` is identity.** For authored content it identifies
  the *kind* (`/stuff/idea/exits/oak-door` — one identity, N anonymous
  clones). For minted singletons it identifies the *instance*
  (`/platform/idea/party/<uuid>`, `/platform/agent/Avatar/<playerId>`). When an instance of
  a multi-instance kind needs its own address, the instance key is
  **scheme-derived**, never random-per-need: the dorm extent slot
  (`…/dorms/f<n>-r<p>`), the persistence spine's uniform
  `(scope, key)`, an exit's relational `<sourcePath>#exit:<dir>`
  (the concealment discovery key). New schemes follow those exemplars.
- **A template *row* is a hydration source, and only authored content
  gets one.** The `domain` collection is the protowizard-editable
  authoring space — nothing else belongs in it. A minted identity
  picks its durable backing by need:
  - **`holder_snapshots`** (the persistence spine) when the thing is a
    persistence host — runtime state, gated writer, never hand-edited
    (Avatars, dorm rooms).
  - **A purpose `Document`** when consumers need queryable fields a
    snapshot can't index (`PartyRecord.memberIds`).
  - **Nothing** when it's transient (`_eval`, `LightningStrike`).

  A per-instance row in `domain` is the **anti-pattern** — it parks
  runtime data in the CMS tree where hand-edits bypass call security.
  The per-player `/platform/agent/Avatar/<playerId>` row is the legacy instance
  (pre-spine); its retirement is tracked work, and no new code should
  copy it.

**Template inheritance does not exist.** The near-precedents are
path-*ancestry* as taxonomy (species clades read the path chain, not
field data) and `Zone.lookupField` (field inheritance through the
*zone* tree). A child template does NOT inherit a parent template's
fields — don't author as if it does, and don't fake it locally; if
real template-data inheritance is ever wanted, it's a deliberate
platform feature, not a per-subsystem hack.
