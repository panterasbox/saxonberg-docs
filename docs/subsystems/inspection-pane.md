# Inspection pane

The persistent right-column cockpit surface that displays what the
player is currently **focused** on. Sourced from two long-lived
MQL subscriptions — a `$focus`-bearing query for the focused-thing
body and a `here`-bearing query for the breadcrumb root — the
pane composes a live-updated header (focus display name), a
paint/clear-gated body (detail when single-focus, list when
multi-focus), a unified breadcrumb that combines the player's
current location, past focus shifts, and any active detail drill,
plus a Refresh button.

Pane policy is **paint/clear**: focus changes clear the body to a
placeholder; an explicit `look` against the current focus paints
it from the live subscription record. The substrate is policy-
agnostic — paint/clear is a client concern that exists to teach
the verbs: *focus is a pointer; look is the verb that paints what
the pointer points at.* A first-delivery auto-paint exception
fires on fresh-session mount so the pane lights up without
requiring an explicit `look` from the player.

See:

- `docs/subsystems/mql-subscription.md` — the substrate this build
  consumes and extends (`mql-query` one-shot, the
  `focusDependent` / `locationDependent` flags, `primaryKeyword` +
  `contents` field-set extensions).
- `docs/subsystems/messaging.md` — `MarkupAugmenter` (in `api/mml.ts`)
  is the pipeline `VisibleMixin.getMarkupLong(viewer)` walks; this
  is how detail keys auto-wrap in the long description shipped on
  every detail projection.
- `docs/subsystems/command-routing.md` — phase-effect option
  declarations (`effects: [{phase: 'focus-update', action: 'skip'}]`)
  back the `look --peek` flow.
- `docs/subsystems/prompt.md` — the prompt's focus token; the pane
  header mirrors it visually but reads from the subscription, not
  the prompt push.
- `docs/slates/tails/client-cockpit-slate.md` — the cockpit layout the
  pane slots into.

## File layout

| File | Role |
|---|---|
| `packages/server/src/mud/api/mql-subscription.ts` | `SubscribeRequest` shape (incl. `focusDependent` / `locationDependent`), `handleSubscribe`, `handleQuery`, `REF_FIELDS` / `DETAIL_FIELDS` extensions |
| `packages/server/src/mud/lib/description/Perceptible.ts` | `primaryKeyword` persistent field, `getPrimaryKeyword` / `setPrimaryKeyword`, fail-soft pool validation, `PerceptibleMixin.subscribableFields` descriptor |
| `packages/server/src/mud/lib/spatial/Container.ts` | `contents` descriptor on `ContainerMixin.subscribableFields`, per-viewer visibility projection, `FieldChangedEvent { field: 'contents' }` fires from `addContainable` / `removeContainable` |
| `packages/server/src/mud/lib/spatial/Containable.ts` | `FieldChangedEvent { field: 'container' }` fires from `setContainer` — the load-bearing signal `locationDependent` subscriptions wake on |
| `packages/server/src/mud/lib/command/Focused.ts` | `setFocus` / `clearFocus` fire `FieldChangedEvent { field: 'focus' }`; `subscribableFields` declares the `focus` descriptor so the index entry installs |
| `packages/server/src/mud/lib/description/Visible.ts` | `getMarkupLong(viewer)` — runs the long description through every contributing mixin's `markupAugmenters` |
| `packages/server/src/mud/lib/description/Detailed.ts` | `wrapDetailKeysAugmenter` contributed via `DetailedMixin.markupAugmenters` — wraps canonical detail keys in `<detail>` MML inline |
| `packages/server/src/mud/lib/boundary/Exitable.ts` | `exits` descriptor on `ExitableMixin.subscribableFields` — ships direction + door affordance for the pane's exit block |
| `packages/server/src/mud/cmd/perception/find.yaml` | `find` verb YAML view (snapshot enumeration, no `updates_focus`) |
| `packages/server/src/mud/obj/command/perception/FindController.ts` | `find` controller — renders one MML row per match, admin viewers see template-path suffix |
| `packages/server/src/mud/cmd/perception/look.yaml` | `--peek` option declares `effects: [{phase: 'focus-update', action: 'skip'}]` |
| `packages/server/src/mud/api/command.ts` | Phase / effects vocabulary (`COMMAND_PHASES`, `PhaseEffect`, `validatePhaseEffect`, `collectPhaseEffects`, `consumePhaseEffects`); dispatcher consults it at the focus-update site |
| `packages/server/src/mud/api/mml.ts` | `MarkupAugmenter` type + `augmentMarkup` helper — substrate for `getMarkupLong` and future inline-affordance pipelines |
| `packages/server/src/mud/api/mixin.ts` | `MixinApi.getAllMarkupAugmenters` — prototype-chain walker the augmenter pipeline consumes |
| `packages/types/src/index.ts` | `MqlQueryMessage` / `MqlQueryResultEnvelope` / `MqlQueryErrorEnvelope` wire types; `StuffRefRecord.primaryKeyword`; `focusDependent?` / `locationDependent?` on `MqlSubscribeMessage` |
| `packages/client/src/store/index.ts` | Pane slice (paint flag, breadcrumb root, breadcrumb trail, detail path, door context, focus name, last result), stuff-registry slice (`Map<stuffId, StuffMetadata>`), `upsertStuffMetadata` |
| `packages/client/src/services/websocket.ts` | `subscribeMql(spec)` / `unsubscribe`, subscription envelope routing, recursive ref-walk that feeds `upsertStuffMetadata` |
| `packages/client/src/components/InspectionPane.tsx` | Pane component — unified breadcrumb (root + trail + detail segments), header, paint/clear body, Refresh, door-context exit synthesis |
| `packages/client/src/components/ui/` | Shared cockpit primitives: `<List>` / `<ListItem>` / `<EntityName>` / `<Button>` + semantic theme `tokens` (see "Shared UI components and theme tokens" below) |
| `packages/client/src/components/MmlRenderer.tsx` | `commandFor()` extended to `<item>` / `<name>` / `<location>` / `<object>` + `<detail>` — registry lookup → `look <primaryKeyword>`, label fallback |

## The pane's two subscriptions

The pane mounts two long-lived MQL subscriptions through the
wire client's `subscribeMql(spec)` method — raw specs, no
indirection layer:

```ts
// Focused-thing body
websocketClient.subscribeMql({
  query: '$focus',
  cardinality: 'many',
  fields: 'detail',
  focusDependent: true,
});

// Breadcrumb root (current location)
websocketClient.subscribeMql({
  query: 'here',
  cardinality: 'one',
  fields: 'ref',
  locationDependent: true,
});
```

The substrate accepts the spec verbatim; there's no server-side
registry of "named subscriptions." Each spec lives in the client
that issues it, and the wire client replays it on reconnect.

### `focusDependent` and the holder-level focus dependency

For a query like `$focus`, the result set is whatever the focus
fragment resolves to — NOT the `FocusedMixin` host. The natural
descriptor walk (which iterates `collectSubscribableFields(stuff)`
for each Stuff in the result set) would miss the focus
dependency entirely.

The `focusDependent: true` flag tells the substrate to install
an additional `(FieldChangedEvent.KIND, 'field', 'focus')`
dependency entry against the **subscription holder** at subscribe
time, in addition to the per-result-Stuff descriptor walk. When
`setFocus` / `clearFocus` fires `FieldChangedEvent { field:
'focus' }` on the holder Avatar, the index entry matches, the
subscription marks dirty, re-resolve runs against the (now-
updated) `$focus` fragment, and the diff produces a delta.

The flag is meaningless for `mql-query` one-shot reads (no
subscription state to wake) and is not carried on the
`MqlQueryMessage` shape.

### `locationDependent` and the holder-level container dependency

Parallel to `focusDependent`. Installs the dep entry
`(FieldChangedEvent.KIND, 'field', 'container')` against the
holder so the subscription wakes on `Containable.setContainer`
fires — i.e., when the player walks, teleports, boards, or
disembarks. The pane uses it to keep the breadcrumb root
synchronized with the current room; without the flag, walking
into a new room would not trigger a re-resolve of `here`.

### `$focus` and `here` at re-resolve time

The substrate runs `ShellApi.expandVariables` against the holder
before each (re-)resolve. For the focus subscription, `$focus`
expands to the holder's current focus fragment fresh on every
tick — that's what makes the `setFocus` → dirty → re-resolve
cascade work end-to-end. The location subscription uses the
built-in MQL pronoun `here`, which resolves to the command
giver's container without any synthetic-var or permission
elevation.

### Single-cardinality slot replacement

The location subscription is `cardinality: 'one'`. When the
player walks from room A to room B, the substrate's diff
produces a single `op: 'replace'` carrying the *new* stuffId —
the old slot is implicitly evicted. The pane's delta handler
consumes the replace op directly rather than running it through
the generic `applyChanges` helper (which keys by stuffId and
would append a duplicate). The flat-cardinality `me.focus`
projection uses `applyChanges` normally.

## Focus-change signaling

`FocusedMixin.setFocus(fragment)` and `clearFocus()` both fire
`FieldChangedEvent { field: 'focus' }` via the substrate's
`MqlSubscriptionApi.fireFieldChange` helper (same pattern as
`NamedMixin.setName`, `VisibleMixin.setShortDescription`). The
helper's strict-equals short-circuit suppresses no-op
emissions — setting the same focus twice fires once.

`FocusedMixin.subscribableFields` declares a `focus` descriptor
purely so the substrate's dependency index installs the
`('stuff.fieldChanged', 'field', 'focus')` entry. No v1 client
field-set asks for `focus` directly; the descriptor exists for
its side-effect on the index. Without it, `setFocus` fires
events into the void.

The alternatives (a purpose-built `FocusChangedEvent` class, or
modeling focus as a per-Interactive observable in the dependency
index) were rejected because the existing `fireFieldChange`
plumbing handles this exact shape with one line on the setter
and one descriptor on the mixin — zero substrate-level changes.

## `PerceptibleMixin.getPrimaryKeyword()` surface

```ts
getPrimaryKeyword(): string | undefined;
setPrimaryKeyword(value: string | undefined): void;
```

A persistent `primaryKeyword` field on `PerceptibleMixin`
(added to a `persistent` entry in `fieldMeta`). The **primary keyword** is
the *guaranteed-resolvable handle* an MML affordance can click —
`look <primaryKeyword>` is the canonical disambiguator.

**Default behavior**: when unset, `getPrimaryKeyword()` returns
`keywords[0]` (the first entry in the derived pool — typically
the first authored keyword, falling back to the first tokenized
name word). When the authored value is set and validly in the
pool, the getter returns it. When the pool is empty (no name,
no authored keywords) and no authored override is set, returns
`undefined`.

**Fail-soft validation on the setter**: the value must appear in
the derived `keywords` pool. Invalid values are ignored with a
warning (so authors can iterate keyword sets without crashes);
state is not corrupted. The getter never calls the setter — they
are strictly independent (a stale invalid stored value is
silently shadowed by the derived-pool head).

**Set-fires-field-change**: real changes route through
`MqlSubscriptionApi.fireFieldChange(this, 'primaryKeyword', ...)`
so subscriptions on `'primaryKeyword'` (any ref record on a
Perceptible host) wake.

## `REF_FIELDS` extension: `primaryKeyword`

```ts
export const REF_FIELDS: FieldSet = [
  'displayName',
  'quantity',
  'primaryKeyword',
];
```

Every ref record shipped by the substrate carries `primaryKeyword`
for Perceptible hosts. Non-Perceptible hosts return `undefined`
from the descriptor and the substrate omits the field on the wire
(same as `quantity` for non-Globbable hosts).

The descriptor lives on `PerceptibleMixin.subscribableFields` —
contributed by the mixin that owns the gate, per the rule
`Stuff.subscribableFields` documents. Non-Perceptible hosts
contribute no descriptor; the substrate's projection loop tolerates
the absence and the field is naturally omitted from those hosts'
wire records (same shape `quantity` uses on Globbable). `REF_FIELDS`
can list `'primaryKeyword'` unconditionally because the loop's
`if (!d || !d.read) continue;` skip handles missing descriptors.

`dependsOnFields: ['primaryKeyword', 'name', 'shortDescription']`
— the getter result changes when any of the derived-pool inputs
change. `changes: [{ on: ShadowChangedEvent, by: 'target' }]` —
keyword pool can be reshaped by shadows.

## `DETAIL_FIELDS` extension: `contents`

```ts
export const DETAIL_FIELDS: FieldSet = [
  'displayName',
  'quantity',
  'primaryKeyword',
  'shortDescription',
  'longDescription',
  'details',
  'bulkMaterial',
  'mass',
  'contents',
];
```

> ⚠ **`details` is per-viewer too, and was the widest identification
> leak in the codebase.** Its descriptor read `(stuff)` while its
> neighbour `longDescription` read `(stuff, viewer)` — so the pane
> enumerated every detail key *and* description regardless of what the
> reader knew, and a detail key names the part by what it does
> (`sigil`, `scorch`). `look` at least had to be asked a question; the
> pane just handed it over. Both layers (`read` and `perDetailRead`)
> now pass the viewer they were always given, through to
> `Detailed.detailRoot(viewer)`. See
> [magic-items.md](./magic-items.md) § *The parts leak harder than the
> prose*.

The `contents` descriptor lives on `ContainerMixin.subscribableFields`.
For container hosts, it ships an array of `'ref'`-shape records
(via `projectFields(child, REF_FIELDS, viewer)`) for visible
contained Stuff. Non-container hosts return `undefined` from the
descriptor and the substrate omits the field on the wire.

**Per-viewer visibility filter**: the descriptor walks
`host.getContents()` and excludes anything the viewer's `Sensor`
perception check rejects (sensory occlusion, etc.). Adornments
and the actor (self) are excluded. The viewer is the
subscription holder threaded through the substrate's
`#projectStuff` pass.

**Containment add/remove diffs** ride
`FieldChangedEvent { field: 'contents' }` fires installed
inline on `ContainerMixin.addContainable` and `removeContainable`.
No new event class, no specialized add/remove diff shape — when
the field-change fires, the substrate re-projects the host, the
new `contents` array goes through the diff machinery, and the
client receives an `op: update` change carrying the patched
list. The cycle is end-to-end via the existing primitives.

The choice to put `contents` directly into `DETAIL_FIELDS`
(rather than a new `'detail-with-contents'` alias or a secondary
subscription on `things in $focus`) trades a minor inefficiency
on non-container detail subscriptions for a uniform projection
policy. If contents grow heavy enough that this matters, that's
the moment to split.

## Paint/clear policy

Client-side; the substrate is policy-agnostic. The pane's body is
gated by `paneBodyPainted` (a flag on the inspection-pane Zustand
slice):

- **On mount** — initially cleared; the placeholder text is
  cardinality-adaptive (single: *"focused — `look` to inspect"*;
  multi: *"N <summary> focused — `look` to list"*) and clickable
  (sends `look`).
- **First-delivery auto-paint exception**: the very first
  non-empty `me.focus` subscription result on a fresh session
  flips `paneBodyPainted = true` immediately. On a fresh login
  the player would otherwise sit on the placeholder until they
  typed `look`; auto-paint elides that step without changing
  the focus-shift-clears rule for subsequent deltas.
- **On focus change** (incoming subscription delta where the
  focused stuffId differs from the prior cached one): clear the
  detail-drill stack; the door context drops if it pinned to the
  prior focus.
- **On `look` against the current focus** (player typed `look`
  with no target, or `look <X>` where `<X>` matched the focus):
  set painted = true, capture the most recent subscription
  result snapshot to `paneLastResult`, render from it.
- **While painted**: deltas update `paneLastResult` and the body
  re-renders in place (React's natural diff). Containment add /
  remove diffs patch the contents list without re-painting the
  rest of the body.
- **While cleared**: deltas update `paneFocusName` (header
  tracks live focus) and `paneLastResult` (cache stays warm)
  but the body stays in placeholder mode.

The two paths to "the body just changed" — outbound `look` paint
toggle and inbound subscription delta — compose: the command
sent → painted = true → delta arrives → result captured → body
renders. The fragment-change-clears-the-body rule applies to
`focus` verb usage, not `look` verb usage (because `look` is
itself the verb whose semantic is *paint the body*).

Pedagogically this matters: auto-painting on focus change would
blur `focus` and `look` into "the thing that updates the pane"
and erase the lesson at the moment players are most likely to
internalize the model. Keeping them visibly distinct teaches the
verb pair.

## Body discipline: percepts, not state dump

The pane body renders **what a perception verb would reveal to
this viewer**, not the focused thing's internal state. `look` is
vision — it reveals appearance and gross features; an estimate at
best for hidden quantities ("looks warm," never "37.4°C").
Internal properties are not perceivable just because they exist.

This is the inspection-pane reconciliation principle (see the
inspection-pane slate's *Reconciliation note* and the
message-rendering-slate):
every fact has a *revelation condition* — which modality /
instrument / skill reveals it, at what fidelity. The viewer
perceives only the facts whose condition they satisfy. The pane's
v1 surface walks this back to the simplest cut:

- **Player body = percept projection.** The substrate's `'detail'`
  field-set ships only percept-shaped fields (display name, short
  / long description, visible contents — already per-viewer
  filtered server-side in `ContainerMixin.contents`); the renderer
  shows them as the look output. No slot maps, mixin lists, raw
  fields, or property bags surface here.
- **Raw internal state is server-side; the v1 pane has no admin
  surface.** Template path, stuff id, mixin composition, raw JSON
  dump, and `clone` / `reload` / `eval` quick actions all belong
  in a future admin surface — but the substrate doesn't project
  those fields today and no client `isAdmin` flag exists. Until
  both ship, the pane carries no admin block; what authors can do
  is use the typed-command interface (`clone <template>`,
  `reload <template>`) just like any verb.
- **Per-fact revelation gating beyond visible is parked.**
  The sense/modality system (feel/smell/listen as separate
  channels), the magic lens, skill-deepens-perception, and per-
  fact provenance all wait for the perception subsystem; the
  spine (fact → revelation condition) is recorded here for the
  future build, the implementation cut is "visible" only.

### Accumulate vs. latest — v1 ships latest-only

When a viewer performs successive perception acts on the same
focus (look, then measure, then appraise), does the pane show
the *union* of percepts each act has revealed, or just the
*latest* act's output?

**Choice: latest-only.** The pane's `paneLastResult` snapshot is
replaced by each subscription result / delta; there is no per-
fact union across multiple `look` / `examine` / `measure`
invocations. The latest-only path stays internally consistent
because the substrate re-projects the *currently-perceivable*
field set on every re-resolve — what the pane shows is what's
true *now*, from this viewer, by the modalities currently in
play.

Accumulate-per-focus is the natural target once the revelation-
condition spine lands; that work is parked alongside the
sense/modality system. v1 does not block on it; the simpler
shape ships and stays correct for the percepts the substrate
currently projects.

## Cardinality-polymorphic body

Same subscription, same field-set (`'detail'` always). The
renderer branches on result-array length:

- **Single (length 1)**: detail view — display name + long
  description (rendered via `MmlRenderer` so embedded MML
  affordances become clickable) + contents list of clickable
  affordances when the focused host is a container.
- **Multi (length > 1)**: list view — one row per match, each
  row's display name rendered as an `<item>`-affordance.

The substrate doesn't know about this branching. Cardinality-
adaptive projection (ship `'ref'` for multi-focus, `'detail'`
for single) was considered and deferred — `'detail'` always for
v1 in exchange for a single uniform projection policy. The minor
inefficiency on multi-focus is accepted.

## `find` verb

```yaml
verbs: [find]
controller: FindController
args:
  - name: query
    type: objects
    required: true
    greedy: true
    scope: ["$focus", "reachable"]
```

**Snapshot semantics.** `find` resolves the query through the
existing MQL pipeline, ships an MML list to the terminal scroll,
one row per match. No `updates_focus` (the absence is load-
bearing — defaults to `'none'`); the giver's focus is unchanged
after `find`. No subscription is registered; no live updates.

**Admin gating.** For admin / Author viewers (checked via
`MixinApi.isAuthor(commandGiver)`), each row appends the
template path in parens — `brass thermometer (/obj/Thermometer)`.
Non-admins see display name only.

**Discovery.** Contributed to `PerceiverMixin.commandContributions.self`
alongside `look`, `scry`, `locate` — `find` is the
enumeration counterpart to `focus`, both surfaced on the
perceiver's verb set.

**`mql-query` integration.** The player-typed `find` rides the
command bus exactly like any other verb (controller renders
prose, player reads it). The `mql-query` one-shot wire surface
exists in parallel for future programmatic consumers — a widget
issuing a `find`-shape read without going through the command
bus. v1 does not exercise that path; the substrate is in place
for when it does.

## `mql-query` one-shot channel

Wire shape in `@saxonberg/types`:

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

`MqlSubscriptionApi.handleQuery(req: QueryRequest)`:

- Reuses ONLY the parse + resolve + project pipeline.
- NO registration in `#registry`, NO dependency-index entries,
  NO listener installation. This is the "share the pipeline,
  skip the state" pattern.
- Holder and cardinality checks mirror `handleSubscribe` so a
  client's error-handling code can branch by `reason` uniformly
  across subscribes and queries.
- `focusDependent` / `locationDependent` are not carried on the
  query shape (no subscription state to wake).

`Application.processUserMessage` routes inbound `'mql-query'`
messages through `handleQuery` — same shape as the existing
`'mql-subscribe'` route. Server-side programmatic one-shot
reads call `MqlApi.resolveOne` / `resolveMany` + `projectFields`
directly; this surface is the wire-facing channel.

## `look --peek` and the phase-effects substrate

`look.yaml`'s `peek` boolean option declares a lifecycle effect
against the dispatcher:

```yaml
args:
  - name: target
    type: object
    scope: ["$focus", "reachable"]
    updates_focus: extend
    prepositions: [at]
    default: "$focus"
options:
  peek:
    type: boolean
    description: "Render prose without changing focus"
    effects:
      - { phase: focus-update, action: skip }
```

The dispatcher recognizes a small vocabulary of *lifecycle
phases* — points between parse and emit where an option can
attach a `skip` or `replace` action. The vocabulary is declared
in `api/command.ts` (`COMMAND_PHASES`, `HOOKABLE_PHASES`,
`REPLACE_HANDLERS`, `IMPLEMENTED_REPLACE_HANDLERS`, `PhaseEffect`,
`validatePhaseEffect`, `collectPhaseEffects`, `consumePhaseEffects`).
Today `focus-update` is the only hookable phase; `validate`,
`confirm-prompt`, `dispatch`, and `emit-scene` are documented
placeholders that the schema accepts but the dispatcher throws
against until their substrate lands.

When the dispatcher's positional-arg loop hits the focus-update
site, it consults `consumePhaseEffects('focus-update', model,
optionDefs)`. If any active option declares
`{phase: 'focus-update', action: 'skip'}` and is truthy on the
bound model, the focus-update step is bypassed. Pronoun memory
still updates; only the focus chain push is held back.

`LookController.execute` is unchanged — it renders the prose
body and emits the existing Scene the same way it always has.
The "peek doesn't change focus" semantic is the dispatcher's
job, not the controller's.

The generalization (a phase taxonomy plus an option-side
`effects:` declaration) lets future flags reuse the substrate
without inventing new schema fields:

- `--async` → `{phase: 'dispatch', action: 'replace', with: 'deferred-dispatch'}`
- `--explain` / `--dry-run` → `{phase: 'dispatch', action: 'replace', with: 'explain-plan'}`
- `--force` → `{phase: 'confirm-prompt', action: 'skip'}`
- `--quiet` → `{phase: 'emit-scene', action: 'skip'}`

Each lands by making its target phase hookable, not by adding a
new YAML field. See `docs/subsystems/command-routing.md` for the
dispatcher details.

## Client stuff registry

A single Zustand slice on `useStore`:

```ts
interface StuffMetadata {
  stuffId: string;
  displayName: string;
  primaryKeyword?: string;
}

interface StuffRegistrySlice {
  stuffRegistry: Map<string, StuffMetadata>;
  upsertStuffMetadata: (records: StuffMetadata[]) => void;
}
```

**Populated** by every subscription consumer the client owns.
When the wire client receives an `mql-subscription-result` or
`mql-subscription-delta` envelope, it walks every record and
calls `useStore.getState().upsertStuffMetadata([...records])`
before dispatching to widget handlers. Nested ref-shape fields
(currently just `contents`; future `equipped`, `inventory`,
etc.) are recursively walked so the registry picks up every
stuff-id mentioned anywhere in the subscription payload.

**Merge semantics**: fields present in the new record
overwrite; fields absent leave existing values intact. A ref-
only delta does not clobber detail data; a detail delta
upgrades a previously ref-only entry.

**Read by**: `MmlRenderer.commandFor()` at click-resolution
time. The renderer reads the registry directly from the global
store snapshot (`useStore.getState().stuffRegistry.get(stuffId)`)
— not via a React subscription, since the renderer just needs
the snapshot at render time. Re-renders happen naturally when
the parent (terminal, pane body) re-renders.

**No eviction in v1.** Sessions are bounded; the registry is
cheap memory. The "forgotten ref leads to broken click" failure
mode (which eviction would create) is more painful than the
"registry grows unbounded" trade-off (which it prevents).

**The registry is a side-effect cache for rendering metadata,
not a source of truth for client logic.** Widgets that need live
state subscribe for it; they do not query the registry. The
registry's only legitimate reader is rendering paths that need
a per-stuff lookup (currently: `MmlRenderer.commandFor`). If a
widget reaches for the registry to answer "what's in this
container" or "is this thing visible," that's the on-ramp to a
client-side shadow model — stop and have that conversation
explicitly rather than slipping into it by accumulation.

## MML identity-tag rendering

`MmlRenderer.commandFor()` extends to four identity tags:

```ts
case 'item':
case 'name':
case 'location':
case 'object': {
  const stuffId = node.attrs['stuff-id'];
  if (stuffId) {
    const meta = useStore.getState().stuffRegistry.get(stuffId);
    if (meta?.primaryKeyword) return `look ${meta.primaryKeyword}`;
  }
  return `look ${node.label}`;  // label fallback
}
```

**Click target** is `look <primaryKeyword>` when the registry
hits, falling back to `look <node.label>` when the registry
misses or the stuff has no primary keyword. Pedagogically this
matters: `look thermometer` is what the player would type.
Showing them that command on hover and sending it on click
teaches the typed-command surface; sending an opaque `stuff-id`
would not.

**Hover preview** rides the existing `onCommandPreview` surface
unchanged.

`<direction>` and `<speech>` remain non-actionable (no
`commandFor` branch). `<exit>` continues to emit `go <dir>`.

**MML identity tags carry `stuff-id` only.** Per the architectural
rule this build establishes, do not add per-stuff attributes to
`<item>` / `<name>` / `<location>` / `<object>` beyond
`stuff-id`. New per-stuff metadata is a projection field on a
subscription, not a tag attribute.

## Shared UI components and theme tokens

The pane composes from a small **shared** primitive set under
`packages/client/src/components/ui/`, not pane-private styled
divs. Three rules govern it:

1. **Reusable primitives, not bespoke JSX.** Future cockpit
   widgets (inventory, self-state strip, group windows) compose
   from the same surface — and so does the layout-MML library
   the message-rendering-slate
   schedules for its Wave 2. When that lands, its `<table>` /
   `<list>` / `<field>` tags map onto these same React
   components, so the subscription-driven rendering path and the
   message-rendered path converge on one DOM shape. The pane
   does not block on that library; it ships its own
   subscription-record → React-component path today.

2. **Semantic DOM = the flatten-linear-labeled floor.** Every
   primitive renders the real HTML element: `<List>` is a `<ul>`
   / `<ol>`, `<EntityName>` is a `<button>`, `<Button>` is a
   `<button>`. No ARIA props are needed to fake what the
   platform already announces. Visual-only `<div>` grids are
   the smell to avoid.

3. **Theme tokens; no hex literals.** All color / spacing / font
   values come from `tokens.ts` — semantic names (`surface`,
   `fg`, `accent`, `border`, `sectionLabel`) that a theme can
   swap wholesale. There is **no** `<color>` or `<size>` MML
   tag; coloring is a stylesheet rule keyed off semantic markup
   (the principle from the message-rendering slate).

| Primitive | Role | Renders |
|---|---|---|
| `<List>` / `<ListItem>` | semantic sequence | `<ul>` / `<ol>` + `<li>` |
| `<EntityName>` | clickable name carrying `stuff-id` | `<button data-stuff-id="...">` |
| `<Button>` | action target with `primary` / `action` / `ghost` variants | `<button>` |
| `tokens` | semantic theme values (color / space / font / radius) | `as const` exports |

### `stuff-id` is double duty: interactivity and styling

`<EntityName>` emits a `data-stuff-id` attribute on the rendered
button. The same attribute drives **two** layers from one source:

- **Interactivity** — the click target resolution layer maps
  `stuffId` (via the stuff registry's `primaryKeyword`) to the
  command this affordance sends. `MmlRenderer.commandFor`
  applies the same registry-then-label fallback for identity
  tags; the pane mirrors it for contents-list rows and multi-
  focus rows via the parent's `onSendCommand` sink.
- **Styling** — a future theme stylesheet selects on
  `[data-stuff-id]` against the viewer's social-graph bucket
  (friend / foe / self) to colour the name. The
  social-graph slate +
  message-rendering slate
  describe the bucket model. The attribute is emitted today;
  bucket selectors land when that subsystem does, without any
  pane changes.

One attribute, two duties — that's the slate's economy. **There
is no `<color>` or `<size>` MML tag**, and no per-tag color
attribute; coloring is always a stylesheet rule keyed off
semantic markup.

### Multi-focus rows: groups, eventually

Per [grouping.md](./grouping.md), the multi-
cardinality `$focus` result is in principle a **group** — `focus
friends` resolves a group via `GroupApi`; the pane renders its
members. v1 has neither `GroupApi` nor friend/foe bucketing, so
the row shape is just "a list of styled names" — and that's the
shape it stays. When `GroupApi` lands, the pane resolves the
group server-side via the same `$focus` subscription and the row
component (`<EntityName>` already carrying `stuff-id`) absorbs
the bucket selector without further work.

## Breadcrumbs and Refresh

### Unified breadcrumb

The breadcrumb is a single horizontal strip combining three
sources of context:

1. **Root segment** — the player's current location, sourced
   from the `here` (`locationDependent`) subscription. Movement
   reroots the strip; the room is always the first segment.
2. **Trail segments** — past focus shifts since the last reroot.
   Pushed by `applyOutgoingCommandToPane` (in `App.tsx`) when an
   outgoing `look <target>` or `focus <target>` doesn't match
   the current root. Capped at 6 entries; the head dedups
   against re-clicks of the same target.
3. **Detail segments** — the active detail-drill stack on the
   currently-focused Stuff. Pushed inline when the player clicks
   a `<detail>` MML affordance in the body prose.

Clicking the root re-focuses the room (sends `look <keyword>` and
clears trail + detail). Clicking a trail entry pops everything
past it, clears any detail drill, then re-sends the entry's
stored command. Clicking an intermediate detail segment pops the
detail stack to that level; the leaf detail segment is rendered
as a non-clickable system label (you're already there).

Movement always reroots: when the `me.location` subscription's
single-cardinality slot replaces, the prior breadcrumb trail is
discarded.

### Door context

When the player clicks a door affordance inside an exit
projection (e.g. `the front doors` rendered next to `south` in
the lobby's exits), the click site stashes a
`paneDoorContext: {stuffId, direction}` annotation in the store.
The door's own pane then synthesizes an "Obvious exits:
<direction>" link in its body so the player can walk through
from the inspection view. The annotation clears on the next
focus shift to a different stuffId.

Pure UI sugar — no substrate change. The door Stuff itself has
no notion of "which exit am I"; the client reconstructs the
relationship from the click site that has both pieces in scope.

### Refresh button

In the pane header. Clicks send `look` through the command bus
exactly like any other click affordance — no special API. Stays
enabled in the cleared-body state (that's its primary use).
Clicks queue if a command is in-flight.

## Reconnect behavior

On WebSocket `connection-established`, the wire client replays
every active subscription's stored spec (the full
`{query, cardinality, fields, focusDependent?, locationDependent?,
detailKey?}` shape it was opened with). The server's substrate
ships the initial result. The pane's header populates from the
first record; the first-delivery auto-paint rule fires the body
into the painted state.

The substrate's per-subscription state on the server is rebuilt
fresh on each subscribe; there is no resume / replay shape.
Mid-reconnect message loss is invisible because the initial
result envelope is authoritative.

## What ships unbuilt

Per the closed-scope requirements:

- **Tabs / tab strip.** v1 ships a single focus tab with no
  strip UI.
- **Pinned `find` results in the pane.** `find` renders to the
  terminal scroll; pinning lands with tabs.
- **Display-flag vocabulary on `find`** (`--bare`, `--with
  vitals`, etc.).
- **`find --focus` flag.**
- **Shift-click alternative on multi-row.** Plain click sends
  `look <that>` (drill-in). The leaned-for alternative gesture is
  shift-click → `find <that>`, peeking a member without collapsing
  the multi-focus; parked until players actually want to inspect a
  group member without leaving the group.
- **`<peek>` MML tag for scrollback clicks.** All clickable
  affordances in the terminal scroll route as plain
  `look <X>` (focus-shifting). No "peek by default for
  backscroll" rule.
- **Per-row aspect families** on multi-focus rows (vitals,
  slots, position).
- **DescribeApi v2 affordances.**
- **Inventory widget** and the `inventory` verb migration.
  Separate tandem slice.
- **New MML tag types** beyond the four already-emitted
  identity tags (`<npc>`, `<player>`, `<command>`, `<quantity>`,
  `<focus>`, etc.).
- **Right-click context menus on MML affordances.** Click →
  `look <label>` is the universal default; tag-specific
  alternative actions ship when the broader gesture vocabulary
  lands.
- **Multi-pane / split view.** Single pane only.
- **Persistent breadcrumb history across reconnects.**
- **Animated focus transitions.**
- **Mobile responsiveness.**
- **Other Chunk 2.6 supporting infrastructure** — heartbeat,
  `ShadowChangedEvent` firing, MQL global seeds
  (`online` / `world`), `mql-subscribe-update` with
  `refresh: true`. Only the `mql-query` one-shot ships here
  because `find` motivates it.
- **Cardinality-adaptive projection** — `'detail'` always.
- **Other client-issued subscriptions** — the pane uses two
  (`$focus`, `here`); future panes / widgets will issue their
  own raw specs through `subscribeMql`. There is no server-side
  registry of "named" subscriptions; each consumer owns its
  spec.
- **Sense/modality system** (feel / smell / listen as separate
  perception channels), the **magic lens**, and
  **skill-deepens-perception.** Recorded as the future
  revelation-condition spine; v1's cut is "visible" only. See
  the *Body discipline* section above.
- **Per-fact revelation gating beyond visible.** Each property
  today is either projected into the detail field-set
  (perceivable as part of the look output) or not surfaced.
  Per-fact provenance (which act revealed this fact, at what
  fidelity) ships with the perception subsystem.
- **Admin / author surface on the pane.** No `isAdmin` flag on
  the auth slice today and no `templatePath` / `mixins` /
  `containerPath` projection on the wire. When admin needs
  arrive, they land with verified substrate (descriptor set,
  per-record gating) rather than client-side speculation.
- **Accumulate-per-focus body.** v1 ships latest-only — each
  subscription result / delta replaces the snapshot. The union
  of percepts across `look` / `measure` / `appraise` waits for
  the revelation-condition spine.
- **`GroupApi` wiring on multi-focus rows.** The grouping
  subsystem isn't built; the row shape (`<EntityName>` carrying
  `stuff-id`) is forward-compatible without component changes.
- **Social-graph bucket styling.** `<EntityName>` already emits
  `data-stuff-id`; the friend / foe / self stylesheet rules land
  when the social-graph subsystem does.
- **Channel stylesheets, `<color>` / `<size>` / heavy layout
  tags.** Out of scope per the message-rendering slate's wave
  ordering; the core stays semantic, presentational tags wait
  for opt-in channel scopes.

## Known future considerations

Flag-don't-fix; just record so future debugging knows the
substrate's choices.

### Per-viewer presentation isn't modeled on the client

Disguise / recognition will eventually want different
`displayName` / `primaryKeyword` per observer. The substrate
already projects per-viewer (each subscription's `viewer` is the
holder's Interactive, so wire payloads are already viewer-
specific). The single client-side `stuffRegistry` slice assumes
one viewer per session, which is true for v1's player-only
client; admin spectating, multi-viewer surfaces, or any "see
this through another's eyes" feature will need to revisit the
registry's keying. Likely shape: a `viewerId` axis on the Map
key, or a per-viewer sub-slice that the renderer routes through.

### Last-writer-wins on conflicting records

Two subscriptions could ship records with different `displayName`
for the same stuff-id — legitimately (under per-viewer
projection, recognition state shifting between subscribes) or as
a bug. Today's merge ("fields present overwrite; absent stay")
accepts the first case silently and provides no signal for the
second. Worth knowing when debugging "why does this thing's
name suddenly differ from what I expected." A future contention-
detection pass could log when an upsert overwrites a non-empty
field with a different value; today the registry is intentionally
quiet.

### Eviction policy upgrade path

Sessions are bounded today, so the never-evict policy is fine.
When sessions get long (long-running NPC tutors, persistent
classroom sessions, etc.), eviction will need a strategy (LRU,
reference-counted by active subscriptions, or session-cap). The
upgrade is clean: consumers only call `Map.get`, so any eviction
policy lands behind the existing read shape without changing
call sites. The decision point is when the registry footprint
crosses whatever profiling threshold makes it the next
optimization candidate.

## Build history

Several substrate shapes shifted during MR iteration. Recording
them here so future debugging knows the substrate's choices
weren't always the obvious ones:

- **Canonical-kind registry retired.** The build initially
  shipped a server-side `MqlSubscriptionApi.registerKind` /
  `CanonicalKindSpec` registry that let clients subscribe by
  name (`'me.focus'`). The registry was a pure server-side
  macro over the wire — same bytes, alias-only — so it was
  demolished in favor of clients sending the raw spec
  (`subscribeMql(spec)`). The substrate's `focusDependent` /
  `locationDependent` flags are now part of the request,
  not derived from a registered kind.
- **`me.location` subscription landed.** Added as a second
  client-issued subscription to drive the breadcrumb root
  (separate from the focused-thing body). Required adding
  `locationDependent` to the substrate's `SubscribeRequest`
  shape and firing `FieldChangedEvent { field: 'container' }`
  from `Containable.setContainer`.
- **`primaryKeyword` descriptor relocated.** Originally on
  `Stuff.subscribableFields` with an inline mixin gate
  (`MixinApi.isPerceptible`); moved onto
  `PerceptibleMixin.subscribableFields` per the rule "mixin-
  gated renders go on the mixin that owns the gate."
- **`getMarkupLong` relocated, augmenter pipeline added.**
  Originally on `Detailed` interface/impl with `VisibleMixin`
  duck-typing into it. The method moved to `VisibleMixin` and
  the wrap-detail-keys logic became the first
  `MarkupAugmenter` contribution. Substrate added: the
  `MarkupAugmenter` type + `augmentMarkup` helper in
  `api/mml.ts` and the `MixinApi.getAllMarkupAugmenters`
  prototype-chain walker. Future contributors (exit-direction
  auto-link, language gating, spoiler hide) plug in via
  `static markupAugmenters` on their mixin.
- **`skip_focus_when_option` retired in favor of phase
  effects.** Originally a single-purpose YAML field on
  positional args; replaced by the dispatcher's phase /
  effects vocabulary in `api/command.ts`. The `look --peek`
  YAML now declares
  `effects: [{phase: 'focus-update', action: 'skip'}]`.
- **Unified breadcrumb.** Two parallel strips (top focus
  breadcrumb + in-body detail trail) collapsed into a single
  strip: root + trail + detail segments. The in-body
  `DetailTrail` was deleted.
- **First-delivery auto-paint.** The original spec said "focus
  changes clear the body; explicit `look` paints." On a fresh
  session that left the pane sitting on the placeholder until
  the player typed `look`. First-delivery auto-paint elides
  the cold-start step without changing the focus-shift-clears
  rule.
- **Admin extras removed.** Originally shipped as forward-
  compatible scaffolding (template-path + mixins + container-
  path display, `clone` / `reload` / `eval` buttons). Removed
  because the substrate doesn't project the fields and the
  auth slice has no `isAdmin` flag — the scaffolding was
  permanent dead code. Future admin needs land with verified
  substrate.

Commit range: `41240c7..HEAD` on the `inspection` branch.
