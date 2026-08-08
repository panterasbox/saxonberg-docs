# MQL subscription substrate

Server-side substrate routing **live MQL queries**: a client sends an
`mql-subscribe` message naming a query + the fields it wants
projected; the substrate registers the subscription under a
per-Interactive id, ships the initial result, and re-projects on every
relevant state change until the client unsubscribes or disconnects.

The substrate is the second wire channel sitting alongside the
prose / dispatch-response pipeline. Inbound dispatch lives in
`Application.processUserMessage`; outbound envelopes ride the same
`MessageApi.sendEnvelope` path the dispatch-response framework uses,
so shadow filters and audit observers see subscription traffic the
same way they see prose.

See:

- `docs/subsystems/mql.md` — the underlying query language.
- `docs/subsystems/response-envelope.md` — the wire envelope framing
  this substrate plugs into.
- `docs/slates/tails/mql-subscription-slate.md` — the closed-scope slate
  that drove this implementation.

## File layout

| File | Role |
|---|---|
| `packages/server/src/mud/api/mql-subscription.ts` | `MqlSubscriptionApi` — a thin facade over `MqlSubscriptionRegistry`; descriptor types, projection helpers, `handleQuery`, the stateless `fireFieldChange` / `projectFields` |
| `packages/server/src/mud/obj/MqlSubscriptionRegistry.ts` | `MqlSubscriptionRegistry` (singleton `Idea`) — owns the per-Interactive subscription map, the meta-bus dependency index, the listener refcounts, and the dirty-set scheduler. State lives here so it survives an HMR reload of the Api file. Every method gates to `FromModule(MqlSubscriptionApi)`. |
| `packages/server/src/mud/lib/events/FieldChangedEvent.ts` | Fact-mixin field-change event (DTO) |
| `packages/server/src/mud/lib/events/PropertyChangedEvent.ts` | Property-bag change event |
| `packages/server/src/mud/lib/events/ShadowChangedEvent.ts` | Shadow lifecycle event (declared; firing wires up in a later subsystem) |
| `packages/server/src/mud/lib/events/GenericEvent.ts` | Escape-hatch class-shaped event |
| `packages/server/src/mud/lib/description/Perceptible.ts` | `primaryKeyword` field + `getPrimaryKeyword` / `setPrimaryKeyword`; `primaryKeyword` descriptor on `PerceptibleMixin.subscribableFields` |
| `packages/server/src/mud/lib/spatial/Container.ts` | `contents` descriptor + `FieldChangedEvent { field: 'contents' }` fires on `addContainable` / `removeContainable` |
| `packages/server/src/mud/lib/spatial/Containable.ts` | `FieldChangedEvent { field: 'container' }` fires on `setContainer` — the signal `locationDependent` subscriptions wake on |
| `packages/server/src/mud/lib/command/Focused.ts` | `focus` descriptor + `FieldChangedEvent { field: 'focus' }` fires on `setFocus` / `clearFocus` |
| `packages/types/src/index.ts` | Inbound + outbound wire types (`MqlSubscribeMessage`, `MqlQueryMessage`, `Mql*Envelope`, `StuffRefRecord`, `WireDetailEntry`, `MaterialSummary`, `StuffDetailFocusRecord`, `Change`) |

## Surface

### `MqlSubscriptionApi`

```ts
handleSubscribe(req: SubscribeRequest): void
handleUnsubscribe(interactive: Interactive, subscriptionId: string): void
handleQuery(req: QueryRequest): void
cancelAllForInteractive(interactive: Interactive): void
```

`SubscribeRequest`:

```ts
interface SubscribeRequest {
  interactive: Interactive;
  subscriptionId: string;
  query: string;
  cardinality: 'one' | 'many';
  fields?: string[] | 'ref' | 'detail';
  detailKey?: string;          // focus-mode opt-in (requires cardinality 'one')
  focusDependent?: boolean;    // install holder-level `focus` field dep entry
  locationDependent?: boolean; // install holder-level `container` field dep entry
}
```

`focusDependent` and `locationDependent` are independent opt-in
flags that install extra holder-level dependency entries the
result-set walk wouldn't naturally find — see "Holder-level
dependency flags" below. There is no named-subscription registry;
clients send the raw spec every time, and the wire client replays
the spec on reconnect.

Substrate behavior:

- Duplicate `subscriptionId` per Interactive → emit
  `MqlSubscriptionErrorEnvelope { reason: 'parse', detail: 'duplicate subscriptionId' }`,
  no registration.
- Missing `query` or `cardinality` → `reason: 'parse'`, no
  registration.
- `detailKey` with `cardinality: 'many'` → `reason: 'parse'`, no
  registration.
- Holder missing `CommandGiver` or `Sensor` composition →
  `reason: 'permission'`.
- MQL parse / resolve throw at subscribe time → `reason: 'parse'` or
  `'resolve'` (substrate distinguishes parse vs MQL-permission via the
  `MqlPermissionError` type — the latter becomes `'permission'`).
- Mid-stream resolve throws → emit `reason` matching the throw type,
  auto-cancel the subscription.
- Holder vanishes mid-stream (no `Sensor` / no `CommandGiver`) →
  silent auto-cancel.

### Descriptor mechanism

Classes declare a `static subscribableFields: SubscribableFieldDescriptor[]`
array. Mixin layers declare their state-bearing fields; `Stuff` itself
declares universal cross-cutting renders (`displayName`, future
`articleName`, etc.). The substrate's `collectSubscribableFields(stuff)`
walks the prototype chain via `MixinApi.getAllSubscribableFields` —
`hasOwnProperty`-checking `subscribableFields` at every level, including
`Stuff`. No substrate-private synthetic table; one mechanism, uniformly
declared.

```ts
interface SubscribableFieldDescriptor {
  name: string;
  read?: (stuff: Stuff, viewer: Stuff & Sensor) => unknown;
  perDetailRead?: (
    stuff: Stuff,
    detailKey: string,
    viewer: Stuff & Sensor,
  ) => Partial<StuffDetailFocusRecord> | null;
  dependsOnFields?: string[];   // defaults to [descriptor.name]
  changes?: ChangeSource[];     // non-FieldChangedEvent dependencies
  static?: true;
}

interface ChangeSource {
  on: { readonly KIND: string };  // EventClass (any class with static KIND)
  by: string;                     // payload-attribute name
}
```

Each descriptor describes one wire field:

- **`read`** — flat projection. Invoked when the subscription is in
  flat mode AND `name` is in the requested field-set. Returning
  `undefined` omits the field from the record.
- **`perDetailRead`** — focused-detail projection. Invoked once per
  focus-mode subscription. Returns a partial record carrying this
  mixin's slice for the focus key, or `null` to contribute nothing.
- **`dependsOnFields`** — leaf source field names whose
  `FieldChangedEvent` fires should mark this descriptor dirty.
  Defaults to `[descriptor.name]` (the primitive case: descriptor
  name === setter's `field` discriminator). Derived fields like
  `displayName` declare explicit deps (`['name', 'shortDescription']`).
  When a derived field depends on another derived field, the
  descriptor author declares the **leaf source fields** (the
  closure of leaves) — v1 has no automatic closure walk; do it by
  hand until chain depth makes that annoying.
- **`changes`** — escape hatch for non-`FieldChangedEvent`
  dependencies. Common case: `ShadowChangedEvent` (a shadow
  attaching can change a computed value without firing a field
  change). Substrate installs these entries in addition to the
  field-event entries derived from `dependsOnFields`.

A descriptor without `read` is flat-mode-skipped. A descriptor without
`perDetailRead` is focus-mode-skipped. `static: true` opts out of
dependency-index installation (no re-resolution).

#### Cascade through dependsOnFields

A client subscribing to `['displayName']` only — `setName('Bob')` fires
`FieldChangedEvent { field: 'name' }`. The substrate's index has an
entry at `(stuff.fieldChanged, 'field', 'name')` installed from the
`displayName` descriptor's `dependsOnFields`. The match marks the
subscription dirty; re-resolve recomputes `displayName` → 'Bob'; diff
emits one delta `{ displayName: 'Bob' }`.

A client subscribing to `['name', 'displayName']` — same fire matches
both descriptors' dependency entries. Re-resolve recomputes both;
diff emits one delta `{ name: 'Bob', displayName: 'Bob' }`. The
cascade is explicit on the server, automatic on the wire.

### Field-set aliases

```ts
export const REF_FIELDS = ['displayName', 'quantity', 'primaryKeyword'];
export const DETAIL_FIELDS = [
  'displayName', 'quantity', 'primaryKeyword',
  'shortDescription', 'longDescription',
  'details', 'bulkMaterial', 'mass',
  'contents',
];
resolveFieldSet(undefined | 'ref') === REF_FIELDS;
resolveFieldSet('detail') === DETAIL_FIELDS;
resolveFieldSet(array) === array;
```

`primaryKeyword` lives on `PerceptibleMixin.subscribableFields`
— contributed by the mixin that owns the gate; non-Perceptible
hosts contribute no descriptor and the substrate's projection
loop omits the field naturally. `contents` lives on
`ContainerMixin.subscribableFields` (non-container hosts omit it
on the wire). See the inspection-pane subsystem doc for the
details surface and the per-viewer visibility filter that scopes
`contents` projection.

### Holder-level dependency flags

The substrate accepts two opt-in flags on `SubscribeRequest`:
`focusDependent` and `locationDependent`. Both install an
**extra dependency entry against the subscription holder** at
subscribe time, in addition to the per-result-Stuff descriptor
walk.

#### Why this is needed

For a query like `$focus`, the result set is whatever the focus
fragment resolves to — NOT the `FocusedMixin` host. The natural
descriptor walk (which iterates `collectSubscribableFields(stuff)`
for each Stuff in the result set) would miss the focus dependency
entirely. Same shape for `here` against the `container` field on
`Containable`.

#### `focusDependent`

Installs `(FieldChangedEvent.KIND, 'field', 'focus')` against the
holder. `FocusedMixin.setFocus` / `clearFocus` fires
`FieldChangedEvent { field: 'focus' }`; the holder-level entry
matches; the subscription marks dirty; re-resolve runs against
the now-updated `$focus` fragment; the diff produces a delta.
End-to-end without any synthetic event class. Used by the
inspection pane's `$focus` subscription.

#### `locationDependent`

Installs `(FieldChangedEvent.KIND, 'field', 'container')` against
the holder. `Containable.setContainer` fires
`FieldChangedEvent { field: 'container' }` on walk / teleport /
board / disembark; the holder-level entry matches and the
subscription wakes. Used by the inspection pane's `here`
breadcrumb-root subscription.

Both flags are meaningless for `mql-query` one-shot reads (no
subscription state to wake) and are not carried on the query
shape.

### Shell-variable expansion at re-resolve time

The query string can reference shell variables (`$focus`, etc.);
the substrate runs `ShellApi.expandVariables` against the holder
before each (re-)resolve so the variable expands fresh every
tick. That's what makes a `$focus`-bearing subscription re-
resolve correctly when `setFocus` fires.

### `mql-query` one-shot channel

`MqlSubscriptionApi.handleQuery` exposes a wire-facing one-shot
read that **shares the parse + resolve + project pipeline** with
`handleSubscribe` but skips all subscription state — no
registration, no dependency-index entries, no listener
installation.

Wire shapes in `@saxonberg/types`:

```ts
interface MqlQueryMessage {
  type: 'mql-query';
  queryId: string;
  query: string;
  cardinality: 'one' | 'many';
  fields?: string[] | 'ref' | 'detail';
  detailKey?: string;
}

interface MqlQueryResultEnvelope {
  type: 'mql-query-result';
  frameId: number;
  queryId: string;
  result: (StuffRefRecord | StuffDetailRecord | StuffDetailFocusRecord)[];
}

interface MqlQueryErrorEnvelope {
  type: 'mql-query-error';
  frameId: number;
  queryId: string;
  reason: MqlSubscriptionErrorReason;
  detail?: string;
}
```

`Application.processUserMessage` routes inbound `'mql-query'`
messages through `MqlSubscriptionApi.handleQuery` — same shape
as the existing `'mql-subscribe'` route. Holder and cardinality
checks mirror `handleSubscribe` so a client's error-handling
code can branch by `reason` uniformly across subscribes and
queries.

Server-side programmatic one-shot reads call `MqlApi.resolveOne`
/ `resolveMany` + `projectFields` directly; this surface is the
wire-facing channel. v1 consumers of the substrate include the
`find` verb pattern documented in
[docs/subsystems/inspection-pane.md](./inspection-pane.md) — the
player-typed `find` rides the command bus, while the
`mql-query` channel is reserved for future programmatic widget
reads.

## Two projection layers

The substrate ships both:

**Flat per-mixin (default).** Each contributing mixin's subscribable
surface lands as a separate field on `StuffRefRecord` /
`StuffDetailRecord`. Faithful to the engine's actual data layout —
DetailedMixin's `details` and TangibleMixin's per-key material
overrides are separately stored, share a key vocabulary but not data,
lifecycle, or events. The wire reflects this. Clients that want a
unified per-detail view join by the shared path-key string.

**Focused-detail (drill-in).** When the client subscribes with
`detailKey?: string` in `MqlSubscribeMessage`, the substrate composes
a per-key view across every mixin that contributes per-detail state.
Each descriptor's `perDetailRead` returns a partial; substrate
merges via `Object.assign` into a `StuffDetailFocusRecord`. The shape
is open (`[key: string]: unknown`) — mixins own their slice's field
names; the substrate keeps no central knowledge of detail shape.

A descriptor whose `perDetailRead` returns `null` contributes
nothing to the merge. Subscriptions with `cardinality: 'many'` AND a
`detailKey` are rejected with `reason: 'parse'`.

## Where descriptors live (v1)

| Layer | Flat fields (`read`) | Focus fields (`perDetailRead`) | Dependencies |
|---|---|---|---|
| `Stuff` (universal) | `displayName` (routes `Stuff.getPresentation()`) | — | `dependsOnFields: ['name', 'shortDescription']`; ShadowChangedEvent in `changes` |
| NamedMixin | `name` | — | default `dependsOnFields: ['name']`; ShadowChangedEvent |
| VisibleMixin | `shortDescription`, `longDescription` | — | defaults; ShadowChangedEvent |
| DetailedMixin | `details` (alias-grouped) | `{ ids, description, hasChildren }` | default `dependsOnFields: ['details']`; ShadowChangedEvent |
| TangibleMixin | `bulkMaterial`, `mass` | `{ material }` (prefix-walk at focus key) | defaults for `bulkMaterial` / `mass`; **explicit** `dependsOnFields: ['detailMaterials']` for `detailMaterial` because the descriptor name doesn't match the setter's field discriminator. ShadowChangedEvent on the shadow-aware ones. |
| GlobbableMixin | `quantity` | — | default `dependsOnFields: ['quantity']` |

`DetailedMixin.getDetailEntries(parent?)` returns alias-grouped
top-level entries (a single Detail object addressed by multiple keys
becomes one entry whose `ids` is every alias). `getDetailEntry(key)`
returns the single alias-grouped entry whose Detail covers `key`, or
`null`. Both back the descriptor's `read` and `perDetailRead`.

### Why `displayName` lives on Stuff

`displayName` isn't a mixin's state — it's a render that pulls from
Named's `name` OR Visible's `shortDescription` OR a baked-in
`'something'` default. It applies to every Stuff regardless of mixin
composition. Three observations:

- The concept is **universal**: every Stuff has a renderable identity.
- There's no "what if this Stuff has no displayable identity?" edge.
- `Stuff.getPresentation()` already encodes the fallback chain
  authoritatively.

So the descriptor lives on `Stuff.subscribableFields` directly. The
prototype-chain walk picks it up via `hasOwnProperty` like any
mixin's descriptor; no synthetic table, no overlay step.

Universal cross-cutting renders → `Stuff.subscribableFields`. Mixin-
gated cross-cutting renders → the mixin that owns the gate.

## ⭐ Ledger-derived fields: the live standing figures

`Avatar.subscribableFields` carries `playStanding`, `makeStanding`,
`renown` and `practisingCompetence` — as **structured values** rather
than a sentence.

⚠ **A trait position is deliberately NOT among them.** The engine
derives one, and the data is trivially reachable, but a pinnable "your
most pronounced trait right now" widget is a stat sheet of your own
personality — the thing the psychology slate calls the *unrealistic*
feature, and the thing that would foreclose the vocation built on **you
cannot read yourself; another person can.** Keeping it off the live
dashboard is what keeps that buildable without retrofitting a
permission model. A test asserts no field name matches
`trait|disposition|personality`, so it cannot drift back in because
someone noticed the data was available.

Declared on `Avatar` rather than on a mixin: `lib/renown/`,
`lib/influence/` and `lib/participation/` hold no mixins at all (those
subsystems are Api + logic singleton + collection), so a `StandingMixin`
for five fields on one class would be per-feature minting. If a second
host ever needs them, that is when it becomes a mixin.

**Self-only.** A `standingSubject()` gate returns the durable subject
key only when the viewer *is* the subject; anyone else gets `undefined`
and the fields are omitted. Reading someone else's standing already has
a surface with a redaction model — the `profile` verb — and a second
implementation here would drift from it.

### ⚠ `read` is sync, and that is load-bearing

`practisingCompetence` reads a ledger whose Api is async
(`AdvancementApi.bandsFor`). **Do not widen `read` to return a
promise.** `MqlSubscriptionApi.projectFields` is
sync, and `ContainerMixin`'s own `contents` descriptor calls it from
*inside* its `read` — so a promise would make projection async
**recursively through nested containment**, on the inspection pane's hot
path.

The other three figures are sync for the right reason: they read a
*materialized standing* (`RenownStanding` and friends keep an in-memory
map warmed from mongo). `lib/standing/DerivedStandingCache` gives the
two un-materialized ledgers the same shape without materializing a whole
collection first:

1. `get()` misses, returns `undefined`, and schedules a background fold
   (at most one in flight per subject).
2. `projectFields` **omits** an `undefined` field — so the client shows
   **no value**, never a zero.
3. The fold lands, `StandingWarmedEvent` fires, the subscription
   re-resolves, and the real figure arrives as a delta.

**A figure the server has not computed yet is absent, not zero.** A zero
would be a claim nobody made. A test asserts the key is missing from the
projection so this cannot quietly regress.

After the first fold, each ledger's own append event
(`lib/events/StandingAppendedEvents.ts`) keeps the cache fresh.

### ⭐⭐ `durableKey` — the witness for ledger-keyed figures

**A `ChangeSource` cannot express these, and quietly pretends it can.**
`deriveAndInstallDependencies` handles `by: 'target'` (indexes
`stuff.stuffId`) and `by: 'field'` (indexes the field name); **anything
else is indexed under the value `null`**, while `routeFire` looks up
`payload[by]`. So a source like `{ on: SomeEvent, by: 'subject' }`
registers under `null`, is looked up under `'/obj/Avatar/dev'`, and
**never matches**. It is not over-eager — it never fires at all. This
build shipped that mistake in its first cut and the tests, which
asserted only that a change source was *declared*, could not see it.

The cause is a genuine impedance mismatch: the bus indexes **live
`stuffId`s**, and every standing ledger keys on the **durable
`templatePath`**. The two cannot meet through a `ChangeSource`.

So a descriptor whose value comes from a durable-keyed ledger declares:

```ts
durableKey: (stuff) => stuff.getTemplatePath() ?? undefined,
```

and its producer calls `MqlSubscriptionApi.notifyDurableSubject(key)`
straight after persisting.

**No event class, no `EventApi` fire.** The bus is for genuinely global
broadcast with unknown consumers; this is one known producer poking one
known consumer, which the codebase's rule makes a method call. It also
matches the **exact** subject, so one player's renown append no longer
wakes every other player's standing pane.

Mechanically it reuses everything: the tuple is installed under a
synthetic `DURABLE_SUBJECT_KIND` so the index, the refcount and the
teardown path all work unchanged — the only special case is that this
kind acquires **no bus listener**, because `notifyDurableSubject` routes
into `routeFire` directly.

`durableKey` returning `undefined` (a guest, with no durable identity)
installs nothing, so nothing is pokeable and nothing throws.

## Event-class pattern

A small class-per-event vocabulary sits on top of the existing
`EventApi.emit(name, payload)` / `on(name, listener)` surface.
Concrete event classes don't share a base — they satisfy the
structural `BusEvent<P>` contract (declared in `api/event.ts`) by
exposing `kind: string` and `payload: P` directly:

| Class | KIND | Discriminator | Firing site |
|---|---|---|---|
| `FieldChangedEvent` | `'stuff.fieldChanged'` | `field` (mixin-declared field name) | Fact-mixin setters via `MqlSubscriptionApi.fireFieldChange` or inline |
| `PropertyChangedEvent` | `'stuff.propertyChanged'` | `property` (property-bag key) | `PropertiedMixin.setProp` |
| `ShadowChangedEvent` | `'stuff.shadowChanged'` | `cause`, `shadow` | **Declared but unfired.** Wires up with the shadow lifecycle in a later subsystem. |
| `GenericEvent<P>` | per-instance | — | Escape hatch when no specific class fits. Has no static `KIND`; routes via string-keyed `EventApi.on(name, …)` only. |

The class-based call site is sugar over the string-keyed bus:

```ts
EventApi.fire(new FieldChangedEvent({ target, field, oldValue, newValue }));
EventApi.on(FieldChangedEvent, (event, ctx) => { /* event.payload typed */ });
```

The class-based `on` routes via the class's static `KIND` string and
delivers a `{ kind, payload }` event-like object to the listener (not
the reconstructed class instance — listeners pattern-match on payload
fields). String-keyed listeners see class-fires; class-keyed listeners
see string-emits to the same KIND. One unified pub/sub bus.

The two namespaces stay separate. A fact-mixin field named `'name'`
and a property-bag key named `'name'` do not cross-trigger — the
meta-bus index discriminates by `(EventClass.KIND, attribute, value)`,
and `FieldChangedEvent` carries `field` while `PropertyChangedEvent`
carries `property`.

### `MqlSubscriptionApi.fireFieldChange` helper

```ts
setName(value: string): void {
  this.name = MqlSubscriptionApi.fireFieldChange(
    this, 'name', this.name, value,
  );
}
```

Strict-equals (`Object.is`) compares old vs new. On equal, skip
emission and return `oldValue`. On change, fire the event and return
`newValue`. Setter is a single line.

The helper lives on `MqlSubscriptionApi`, not on `FieldChangedEvent`,
so the substrate owns its own helpers and the event class stays a
pure DTO.

Used by `NamedMixin.setName`, `VisibleMixin.setShortDescription` /
`setLongDescription`, `GlobbableMixin.setQuantity`. `DetailedMixin`
(setDetail / removeDetail), `TangibleMixin` (setMaterial / setMass),
and `PropertiedMixin` (setProp) inline the fire because their
mutation shape doesn't fit the helper cleanly (Map mutations,
multiple-field changes, marshaller boundary).

## Meta-bus dependency index + scheduler

`MqlSubscriptionRegistry` maintains a 3-level Map keyed by
`(EventClass.KIND, attribute-name, attribute-value)`:

```
private index: Map<string, Map<string, Map<unknown, Set<SubscriptionState>>>>
```

(It's `private`, not `#`-private: the Registry is a Stuff host whose
instance methods dispatch through the call-security proxy, where
`#`-private slots are unreachable.)

Field-event dependencies come from `descriptor.dependsOnFields` (or
its `[descriptor.name]` default). The substrate translates each
dep to a `('stuff.fieldChanged', 'field', <dep>)` index entry.
Non-field dependencies (the `changes` escape hatch) install their
own entries — most commonly `('stuff.shadowChanged', 'target',
<stuffId>)` so a shadow attaching to the host re-projects derived
fields without firing a synthetic field event.

One `EventApi.on(KIND, …)` listener is installed per distinct
`(KIND, by)` pair, refcounted across subscriptions. The handler reads
`payload[by]`, looks up the value bucket, walks every subscription in
the set, and calls `markDirty(sub)`.

```ts
private dirty = new Set<SubscriptionState>();
private scheduled = false;
markDirty(sub): void {
  this.dirty.add(sub);
  if (!this.scheduled) {
    this.scheduled = true;
    setImmediate(() => this.drainDirty());
  }
}
```

`dirty`, `scheduled`, `markDirty`, and `drainDirty` all live on
`MqlSubscriptionRegistry` (`private`, per the proxy rule above).

`setImmediate` batches: N events for the same target in one tick
produce ONE re-resolve per affected subscription. The drain runs the
dirty set in insertion order, re-resolves each query via
`MqlApi.resolveOne` / `resolveMany`, diffs against `lastResult`, and
emits a single `MqlSubscriptionDeltaEnvelope` per subscription whose
diff is non-empty. Subscriptions cancelled between dirty mark and
drain are dropped silently.

## Diff algorithm

`Change.op` is one of `'replace' | 'update' | 'add' | 'remove'`.

**Single-cardinality:**

- old null + new top → `op: 'replace'` with full record
- old top + new null → `op: 'remove'`, no `fields`
- different top stuffIds → `op: 'replace'` with full record
- same top stuffId, fields differ → `op: 'update'` with diffed fields

**Collection cardinality:**

- key in new only → `op: 'add'` with full record
- key in old only → `op: 'remove'`, no `fields`
- key in both, fields differ → `op: 'update'` with diffed fields

Equal-record updates (every field deep-equal) skip the change; empty
change lists emit no envelope (silent no-op).

## Disconnect cleanup

`Application.handleUserDisconnect`:

1. Resolves the `Interactive` from the socket id.
2. Calls `MqlSubscriptionApi.cancelAllForInteractive(interactive)`
   **before** `ConnectionManager.removeInteractive(socketId)`. The Api
   delegates to `MqlSubscriptionRegistry.cancelAllForInteractive`,
   which clears the Interactive's entry in the Registry's
   per-Interactive `registry: Map<Interactive, …>` slot. Each
   subscription's dependency-index entries deregister and listener
   refcounts decrement (and `unsubscribe()` when zero).
3. Then `ConnectionManager.removeInteractive` runs.

The ordering keeps the Interactive live for any final substrate-side
envelope delivery during cancellation (silent in v1, but
`reason: 'closed'` envelopes may ship in a later subsystem).

## Conservative-coarse dispatch policy

The v1 dispatcher errs toward more re-resolves rather than missed
updates. Specific shapes:

- **Field-keyed firing is global, not per-target.** A descriptor
  whose `dependsOnFields` includes `'name'` marks every subscription
  on a name-dependent field, regardless of which Stuff actually
  fired the change. The diff pass filters out fireworks that don't
  change a subscription's actual result; the index pre-filter just
  avoids obvious mismatches.
- **Focus subscriptions re-resolve on any matching
  `FieldChangedEvent`, including changes to other detail keys.**
  `TangibleMixin.setMaterial(_, 'tail')` fires
  `FieldChangedEvent { field: 'detailMaterials' }`, which is
  index-matched by any focus subscription on `head.edge`'s
  `detailMaterial` descriptor; the re-projection finds no actual
  change in `material` and emits nothing. Adaptive per-key
  refinement is deferred until profiling demands it.
- **`mass` / `bulkMaterial` are independent.** Each fires its own
  `FieldChangedEvent { field: 'mass' | 'bulkMaterial' }`; the diff
  ensures only the actually-changed field appears in the update.

These over-fire patterns are the right v1 trade. The cost is a few
extra re-resolves on a quiet codepath; the benefit is no risk of a
descriptor missing a notification because the index missed a
discriminator.

## `displayName` routes through `Stuff.getPresentation()`

The universal `displayName` descriptor's `read` delegates to
`Stuff.getPresentation()`:

```ts
{ name: 'displayName', read: (stuff) => stuff.getPresentation(), … }
```

`getPresentation()` is the single source of self-presentation policy:

- **No caller-supplied fallback.** The baked-in `'something'` is the
  bottom of the chain, so the return type stays `string` (never
  null) and the substrate's `StuffRefRecord.displayName` is
  non-optional, honored without coercion.
- **Viewer-aware at the projection seam.** The `displayName` descriptor's
  own `read` stays viewer-blind (`getPresentation()`), but `projectFields`
  renders the universal `displayName` field through
  `RecognitionApi.describe(viewer, stuff)` — so each subscriber sees its
  *own* perceived name (recognition / identification / disguise), the same
  routine the prose path uses. The descriptor isn't overloaded; the
  viewer-aware step is applied at projection. See
  [belief.md](./belief.md).
- **Count folds in.** For a `Globbable` stack `getPresentation()`
  returns `"30 coins"`; the `quantity` field still rides along
  separately for clients that want the raw number.

(Historically this lived on a static `DescribeApi.getDisplayName`
that the subscription reshape took a `viewer?` and dropped a
caller `fallback` arg; that Api retired when self-presentation
moved onto `Stuff`.)

## What ships unfired

`ShadowChangedEvent` ships **declared but unfired**. Descriptors that
participate in shadowed projection (NamedMixin, VisibleMixin,
DetailedMixin, TangibleMixin's `bulkMaterial` / `detailMaterial`, and
the universal `displayName` on `Stuff`) reference it in their
`changes` array so the meta-bus index is wired end-to-end; until the
firing site exists, the listener is silent.

When a shadow lifecycle subsystem lands, firing `ShadowChangedEvent`
from attach / detach / mutate sites automatically lights up the
substrate's re-projection without any descriptor changes.

## What doesn't ship at all (v1)

- `mql-subscribe-update` — change a subscription's query without
  un/re-subscribing.
- Heartbeat / explicit `'closed'` envelope.
- Wire projection fields for `mixins[]` / `capabilities[]`.
- Per-detail-key dependency-index refinement.
- Programmatic / non-client substrate consumers (e.g., server-side
  rules listening for live state). The `mql-query` channel exists
  as a wire surface; in-process consumers call `MqlApi.resolveOne`
  / `resolveMany` directly.
- Server-side named-subscription registry. The build initially
  shipped one and retired it during MR review — clients send the
  raw spec via `subscribeMql(spec)` and replay it on reconnect.
  See *Build history* below.

These land in follow-up scope when concrete consumers demand them.

## Build history

- **Canonical-kind registry retired during MR iteration.** Wave 1
  shipped `MqlSubscriptionApi.registerKind(name, spec)` with a
  `kind?: string` overlay on `MqlSubscribeMessage` /
  `MqlQueryMessage`. The mechanism was a pure server-side macro
  over the raw spec — the substrate's load-bearing machinery
  (dependency-index, diff, listener wiring, the `focusDependent`
  flag) didn't depend on the registry. Demolished in favor of
  clients sending the raw spec; the wire bytes are unchanged for
  any caller that was sending the spec directly.
- **`locationDependent` added.** Symmetric with `focusDependent`,
  installs a holder-level dep entry against the `container` field
  so subscriptions on `here` (or any holder-position query) wake
  on movement. `Containable.setContainer` now fires
  `FieldChangedEvent { field: 'container' }`. Used by the
  inspection pane's breadcrumb root.
- **`primaryKeyword` descriptor relocated.** Originally on
  `Stuff.subscribableFields` with an inline mixin gate
  (`MixinApi.isPerceptible`); moved onto
  `PerceptibleMixin.subscribableFields` so the descriptor sits on
  the mixin that owns the gate. The substrate's projection loop
  already tolerates missing descriptors, so `REF_FIELDS` can list
  `'primaryKeyword'` unconditionally even though only Perceptible
  hosts contribute one.

Commit range: `41240c7..HEAD` on the `inspection` branch.
