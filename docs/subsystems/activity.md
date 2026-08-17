# Activity framework

The activity framework turns durative actions into first-class
runtime objects. An `Engagement` is a thing-in-progress on an actor:
a spell being cast, a sword being forged, a song being sustained,
the act of climbing a face. The framework owns timing, slot
occupancy, observation, transactional commit, and abort discipline
uniformly across all activity shapes.

Wave 1 ships the substrate — `EngagedMixin` on `Character`,
`SchedulerApi` with the five-outcome start contract, the
activity-class registry that makes lifecycle dispatch HMR-aware, the
five framework-intrinsic abort reasons, and the `cancel` verb. **No
v1 controllers register engagements yet.** Walking, mounting,
sitting, reading, etc. stay synchronous — v1's game-feel target is
responsive controllers, not durative content for its own sake. The
framework lands inert, ready for genuinely-durative content
(spellcasting, forging, brewing, ritual, summoning) when it
arrives.

**The first real consumer has since landed:**
[respiration](./respiration.md) is the first concrete engagement
producer — its `RespirationDrain` / `RespirationRecovery`
`SustainedEngagement`s, each with a recurring `ScheduledEmission`
delegating back to the host mixin, are the worked exemplar of the
substrate below (start/cancel, the emission cadence on game-time, the
`getHost` host-destruction hook). The crisis drain occupies the `'body'`
slot; the deferred strangulation channel reserves `'voice'`.

The first *behavior* consumer has also landed:
[NPC behavior](./behavior.md) is the activity substrate's first
non-physiological consumer — `wanders`/`patrols` cadence brains traverse
on a jittered schedule, and the generic `BehaviorBeat` `DurativeActivity`
rides the scheduler purely to occupy an engagement slot for a contention
window (so a wandering NPC yields `'body'`/`'attention'` while it greets
an arrival). See [behavior.md](./behavior.md).

The first *durative-verb* consumer has also landed:
the manual cocktail-build verbs (`pour`/`stir`/`strain`/`garnish`) each
run as a `ManualBuildStep` `DurativeActivity` on the `'hands'` slot — the
worked exemplar of "a verb with duration becomes an activity" (effect
applied **at completion**, a barge-in `cancel` aborts mid-step leaving
partial matter standing). See [crafting.md](./crafting.md) and the paced
replay it enables in [scripting.md](./scripting.md).

Three claims drive the design:

1. **Verbs that have duration become activities.** A controller's
   `execute()` either completes the verb (if instant) or calls
   `SchedulerApi.start()` to register an activity (if durative). For
   the durative case the controller pushes the scheduler's returned
   `EngagementStartedNote` onto its dispatch via `ctx.note(...)` and
   returns `void`. The activity runs on its own scheduler timer; the
   dispatch envelope ships immediately so the connection is ready
   for the next command.
2. **Activities don't block command processing.** An actor with an
   in-flight activity can still type new commands. Whether the new
   command runs depends on its compatibility with the current
   engagement, not on whether the engagement has completed.
3. **Activity completion is a transaction boundary.** The completion
   handler runs synchronously, re-validates preconditions, and
   commits the activity's mutations atomically. Aborts don't roll
   back state — they fire *before* any state change, by design.
   Completion and abort fire long after the originating dispatch
   finished — there is no `CommandContext` in scope; the scheduler
   emits its own `activity-update` envelopes via
   `MessageApi.sendEnvelope`, independent of any command.

## The Engagement interface family

Every actor-occupying thing is an `Engagement` of one of two shapes:
`DurativeActivity` (has a timer, completes on its own) or
`SustainedEngagement` (no timer, ends only when an explicit verb
aborts it). Both shapes occupy engagement slots identically; the
framework treats them uniformly for concurrency and abort
semantics.

```typescript
// api/scheduler.ts

interface Engagement {
  // Identity — framework-stamped at start(); referenced by every wire note.
  engagementId: string;
  readonly type: string;             // 'walk', 'climb-hold', 'read', 'forge'
  readonly actor: Stuff & Engaged;
  readonly startedAt: number;        // wall-clock ms; serializable

  // Concurrency / interruption policy
  readonly slots: ReadonlySet<EngagementSlot>;
  readonly interruptibleBy: ReadonlySet<AbortReason>;
  readonly cancelable: boolean;      // ships on EngagementStartedNote
  readonly emissions?: readonly ScheduledEmission[];

  // Lifecycle
  onStart(): void;
  onAbort(reason: AbortReason): void;
  getHost?(): Stuff | null;          // eager host-destruction hook
}

interface DurativeActivity extends Engagement {
  readonly duration: number;         // ms; values < 100 → completed-sync
  readonly replaceableBy: readonly string[];
  onComplete(): void;
}

type SustainedEngagement = Engagement;
// structurally identical; absence of `duration` / `onComplete` is the discriminator

interface ScheduledEmission {
  intervalMs: number;
  event: (ctx: { engagement, actor, elapsed }) => unknown;
}
```

`type` is the canonical key for `cancel <type>` and (for activities)
replaceable-match. `slots` drives concurrency. `replaceableBy` and
`interruptibleBy` are per-engagement policy. `emissions` is the
declarative cadenced-side-effect channel — authors describe what
fires on what cadence; the scheduler owns the timers.

**Cancelability on the wire.** Every `Engagement` ships with
`EngagementStartedNote.cancelable: boolean`, computed at start time.
v1 default is `true`. The five framework-intrinsic abort reasons
are always accepted, so the `cancel` verb always works against
every v1 activity. Authors who later need a non-cancelable
engagement (a resolved fall, a checkmate animation) set
`cancelable = false` on the subclass.

## Engagement slots — concurrency vocabulary

Slots model "what part of the actor is engaged." An actor can read
while walking (different slots); can't climb while reading (both
take attention).

| Slot        | Engaged by                                          |
|-------------|-----------------------------------------------------|
| `body`      | locomotion, posture changes, mounting               |
| `hands`     | climbing, fine manipulation, multi-handed verbs     |
| `attention` | reading, casting, observing intently, climbing      |
| `voice`     | sustained song, chant (instant `say` doesn't engage) |

v1 vocabulary; expand if a content case demands a finer
distinction. `eyes`-separate-from-`attention` and `posture-lock`
were considered and rejected for v1 — keep the vocabulary small;
split only when forced.

The concurrency rule: two engagements can co-occupy iff their slot
sets are disjoint.

## The Scheduler

> **Non-engagement cadence consumers exist.** Not every cadenced
> side-effect is an engagement. Condition progression (Vitals) occupies
> **no** engagement slot — it borrows the ticking machinery via
> **`ScheduleApi.recurring`** directly, NOT the engagement-bound
> `ScheduledEmission` (whose callback takes `{ engagement, actor,
> elapsed }`). When a recurring side-effect isn't something an actor is
> *doing*, target `ScheduleApi.recurring`. (Vitals ships the progression
> *shapes*; no live ticks yet — see [vitals.md](./vitals.md).)

`SchedulerApi` (`api/scheduler.ts`) is a thin facade; the
`SchedulerRegistry` singleton (`obj/SchedulerRegistry.ts`) owns the
active set of engagements and every timer backing them. The timers are
**game-time** schedules, not raw Node timers: one `WorldClockApi.after`
`ClockHandle` per `DurativeActivity` completion, one
`WorldClockApi.every` `ClockHandle` per `ScheduledEmission` — so
pausing the world clock pauses in-flight activities. Per-engagement
host-destruction subscriptions ride `EventApi.on(Events.StuffDestructed,
...)`. On abort or completion the Registry cancels every clock handle
it owns for that engagement in one place — no per-author bookkeeping.

### Surface

```typescript
class SchedulerApi {
  // Lifecycle
  static start(engagement: Engagement): StartResult;
  static cancel(engagement: Engagement, reason: AbortReason): void;
  static cancelAll(actor: Stuff & Engaged): void;
  static cancelByType(actor: Stuff & Engaged, type: string): void;
  static cancelByPredicate(actor, pred): void;

  // Introspection
  static getEngagements(actor): readonly Engagement[];
  static getEngagementBySlot(actor, slot): Engagement | undefined;
  static getEngagementById(id): Engagement | undefined;

  // Activity-class dispatch index (HMR seam — see below). There is
  // no registration surface: start() captures the engagement's own
  // class (capture-at-start).
  static getActivityClass(type: string): ActivityClass | undefined;
  static reloadActivity(type: string): Promise<void>;
}
```

### The five outcomes of `start()`

`StartResult` is the scheduler→controller handoff for the start
case only. Programmer errors throw; world-state events return.

```typescript
type StartResult =
  | { ok: true; status: 'started' | 'replaced';
      engagement; note: EngagementStartedNote;
      replaced?: readonly Engagement[] }
  | { ok: true; status: 'completed-sync'; engagement }    // wire-silent
  | { ok: false; reason: 'engagement-conflict';
      conflicts: readonly Engagement[] }
  | { ok: false; reason: 'start-rejected'; error: Error }; // onStart threw
```

Controller branch shape:

- `ok:true` with a `note` → `ctx.note(note)` + begin-prose via `Scene.send`.
- `ok:true` `'completed-sync'` → completion-prose only; no `ctx.note`. Wire-silent.
- `ok:false` `'engagement-conflict'` → `ctx.note({kind:'controller-rejected', reason:'engagement-conflict', ...})` + conflict prose.
- `ok:false` `'start-rejected'` → `ctx.note({kind:'controller-rejected', reason:'start-rejected', detail: error.message})` + rejection prose.

### `start()` algorithm

```
SchedulerApi.start(next):
  conflicts ← engagements occupying any slot in next.slots
  if conflicts:
    if !every-conflict.replaceableBy.includes(next.type):
      return { ok: false, reason: 'engagement-conflict', conflicts }
    abort(c, 'replaced')  for each conflict  ← out-of-band envelope each
  stamp next.engagementId via nanoid
  try { next.onStart() } catch (err) { return { ok: false, reason: 'start-rejected', error } }
  if isDurativeActivity(next) && next.duration < 100:
    runOnCompleteInPlace(next)   ← validate + mutate sync; no timer, no envelope
    return { ok: true, status: 'completed-sync' }
  register(next)   ← slot map + by-id index + completion timer + emission timers
                   + host-destruction subscription (if next.getHost?())
  return { ok: true, status: 'started' | 'replaced', engagement, note, replaced? }
```

### Lifecycle dispatch — HMR-aware

Post-construction lifecycle hooks (`onComplete`, `onAbort`,
`getHost?`) route through the activity-class dispatch index, NOT
through the instance prototype. The index populates by
**capture-at-start**: `start(engagement)` records
`engagement.constructor` under `engagement.type` — no registration
step exists (the no-module-scope-statements rule). After a
`SchedulerApi.reloadActivity(type)` (which re-points the entry
explicitly), or simply the next `start` of that type, the index holds
the freshest class. Lifecycle dispatch
is `cls.prototype.onComplete.call(engagement)` — bound to the
instance, but resolved through the latest class. In-flight
engagements pick up newly-reloaded code on their next fire.

`onStart` is the exception — it runs once at start time against
the just-constructed instance, which was constructed from whatever
class was registered at that moment. Direct instance dispatch is
fine there.

`SchedulerApi.reloadActivity(type)` is the wizard-callable
cache-bust: it resolves the activity class's source path via
`ModuleApi.lookup`, strips the `#exportName` suffix, and calls
`HotReloadApi.reload`. Intended for use via the `eval` verb until a
dedicated reload verb ships.

**Registry miss recovery.** If a class is unregistered between
`start()` and a lifecycle fire (file deleted, type renamed,
`SchedulerApi.reloadActivity` torn down mid-flight), the dispatch
helpers log and degrade gracefully:

- `dispatchOnComplete` misses → force `onAbort('thrown')`.
- `dispatchOnAbort` misses → log; timers already cancelled by the
  outer abort path, so the engagement is correctly torn down even
  though activity-specific cleanup didn't run.
- `dispatchGetHost` misses → return `null` (engagement opts out of
  the eager subscription).

(These dispatch helpers — and the `activityRegistry` they read — are
`private` members on `SchedulerRegistry`, not Api `#`-private fields:
the Registry is a Stuff host whose instance methods dispatch through
the call-security proxy.)

### Sub-100ms completion-sync

`duration < 100` triggers in-place synchronous completion. The
scheduler:

1. Runs `onComplete()` in `start()`'s frame (no timer).
2. Skips slot-map registration entirely.
3. Skips the wire envelope entirely (slate: "the activity never
   appeared on the wire as in-flight").
4. Returns `{ ok: true, status: 'completed-sync' }`.

The controller renders completion prose itself. Authors write
`onComplete()` exactly once; the framework picks the
schedule-or-run-now path. A 50ms `KneelActivity` gets correct
synchronous semantics for free.

Rationale: below 100ms the activity-shaped observability buys
nothing the client can perceive (it's below network jitter); above
it, the activity machinery pays back.

### Watchdog discipline

Every `onComplete`, `onAbort`, and emission-fired event runs inside
a try/catch. An exception logs the error, fires
`onAbort('thrown')` to force the engagement into the abort state
with timers cancelled, and ships an `EngagementCancelledNote`
envelope. The server keeps running.

`'thrown'` is wire-visible: clients can surface "something went
wrong" prose differently from a clean player cancel. The recursive
case — `onAbort('thrown')` itself throws — is logged but does not
retry; the engagement is force-removed from the active set and its
timers cancelled.

`onStart()` is the exception. It runs *before* registration, so a
throw isn't a watchdog case — the scheduler hasn't registered
timers yet. The throw converts to
`{ ok: false, reason: 'start-rejected', error }`.

### Timer-callback gating

Timer callbacks (setTimeout / setInterval) fire on Node's event loop
with no calling frame on the call-security stack. The scheduler still
plants a fresh root for context hygiene where a callback fires, but the
privileged `EngagedMixin._setEngagement` / `_clearEngagement` mutators
no longer need a synthetic `SchedulerApi` frame to pass their gate:
they carry the **participant contract**
`FromTemplate('/obj/SchedulerRegistry')` — the registry (the machinery
whose timer set must stay in sync with the map) calls them as itself
from `register`/`deregister`, with its own frame on the stack.

## EngagedMixin

`EngagedMixin` (`lib/activity/Engaged.ts`) is the actor-side
storage seam. Composed on `Character` so every PC and NPC carries
the engagement slot map uniformly. No required co-mixins — a
stationary forge-bound creature can be `Engaged` without being
`Mobile`.

```typescript
interface Engaged {
  getEngagements(): readonly Engagement[];
  getEngagementBySlot(slot): Engagement | undefined;
  getEngagementByType(type): Engagement | undefined;
  hasEngagement(slot): boolean;

  // Privileged — participant-gated: only the SchedulerRegistry
  // singleton may call. Runtime-rejected for every other caller.
  _setEngagement(slot, engagement): void;
  _clearEngagement(slot): void;
}
```

The map is **runtime-only** — `_engagements` is deliberately NOT in
`fieldMeta`'s persistent entries. A reloaded actor wakes up with no live
engagements (no timers to recover, no subscriptions to re-stitch).
Mirrors `Mobile._engagedModePath`'s runtime-only treatment.

`_setEngagement` / `_clearEngagement` are `@Final @Unshadowable` +
participant-gated `FromTemplate('/obj/SchedulerRegistry')` — only the
scheduler registry may mutate the map. The leading `_` flags the
surface as privileged, not for general callers.

EngagedMixin also contributes the `cancel` verb and the `stop`
default alias (collected via `AliasMixin.queryMixins`-driven
default-alias walk on actors that compose both mixins).

## The `cancel` verb

```
cancel             → SchedulerApi.cancelAll(actor)
cancel <type>      → SchedulerApi.cancelByType(actor, type)
stop               → alias for `cancel` (default-aliased by EngagedMixin)
```

`<type>` is a plain string — not an MQL `object` reference. Slate is
explicit: "operates on the actor's `engagements` map, not on world
objects." `cancel reading` matches engagements with `type ===
'reading'`. The activity-type-to-verb mapping is the activity
author's concern (a future `TraverseActivity` would self-register
under `'walk'`, `'climb'`, etc. — see
docs/slates/tails/locomotion-as-activity-slate.md
for the deferred design).

`cancel` itself is instant — no slots, runs synchronously, always
available regardless of what's in flight.

Wave 1 originally shipped an empty engagement map (no activity
producers); engaged consumers have since landed (respiration, behavior,
the manual-build verbs — see above). Bare `cancel` still narrates "You
cancel what you were doing." against an empty map; `cancel <type>` emits
an `empty-result` note + "You aren't <type>." when no matching activity
is in flight.

## The Abort taxonomy

`AbortReason` is distributed via TypeScript declaration merging,
mirroring `MqlMatchVia`. The empty `AbortReasonRegistry` interface
lives in `@saxonberg/types` so the envelope's
`EngagementCancelledNote.reason` typechecks across the package
boundary. Each subsystem that can cause an abort augments the
registry from its own `lib/<subsystem>/abort-reasons.ts` (or, for
small augmentations, inline at the top of the subsystem's primary
mixin file — the activity framework does this in `Engaged.ts`).

The five framework-intrinsic reasons:

| Reason                  | Fired when                                                |
|-------------------------|-----------------------------------------------------------|
| `cancelled`             | Explicit player cancel (`cancel`, `stop`).                |
| `replaced`              | Another engagement preempted this one via `replaceableBy`.|
| `preconditions-changed` | `onComplete` revalidation failed; activity self-aborts.   |
| `host-destroyed`        | The engagement's `getHost()` Stuff was destructed.        |
| `thrown`                | `onComplete` / `onAbort` / emission event threw.          |

Other reasons live in their owning subsystems — `combat`,
`social-interrupt`, `posture-changed`, `environmental`, `restart`
(future shutdown-lifecycle). None belong here; they augment as
their content earns the framework slot.

### Acceptance policy

`interruptibleBy: Set<AbortReason>` is an **opt-in list for
non-framework-intrinsic reasons**. The five framework-intrinsic
reasons are always accepted — you can't opt out of "the framework
is aborting you because you called cancel." Other reasons abort the
engagement only if they appear in `interruptibleBy`.

## Pre-completion mid-flight validity

An activity scheduled at t=0 to complete at t=4000 sits unattended
in the scheduler. The world might change in those 4 seconds. The v1
policy is **lazy revalidation with one specific eager hook**:

- **Lazy default** — `onComplete` is the only general validation
  point. If the world changed in a way that doesn't match
  preconditions when the timer fires, abort with
  `'preconditions-changed'`.
- **Host-destruction hook (eager)** — when an engagement returns a
  Stuff from `getHost()`, the scheduler subscribes via `EventApi`
  to `Events.StuffDestructed` for that stuffId and fires
  `onAbort('host-destroyed')` immediately on match. Cheap (no
  polling); catches the most-jarring failure mode (the horse you
  were mounting is killed, the ladder you were climbing rots away).

Future per-activity opt-in to broader eager revalidation can land
when a specific case earns it (cast a spell on a target — if the
target dies mid-cast, fail early to avoid wasting mana). v1 ships
only the host-destruction hook.

## Transaction-style completion

The completion handler is the transaction boundary:

1. **Re-validate preconditions** — slot still available, target
   still reachable, actor still where they were when the activity
   started, mode still eligible.
2. **If anything's changed** → call
   `SchedulerApi.cancel(this, 'preconditions-changed')` and
   `return`. (The scheduler's cancel path fires `onAbort` and
   handles deregistration; don't throw, don't run mutations.)
3. **Otherwise** → execute mutations synchronously.
4. **Emit wire frames** — completion prose `MessageFrame`s for
   actor + peers; `activity-update` envelope with
   `EngagementCompletedNote`; mutations produce state-sync deltas
   through the standard channel.

No `await` anywhere in the body. The atomicity is "no other JS code
interleaves while `onComplete` runs" — *not* "throws roll back side
effects." The validate-then-mutate ordering (steps 1–2 before steps
3–4) is what keeps "completion is the commit point" honest in
practice.

The activity is essentially a deferred, scheduled mutation; when
its timer fires, the mutation either applies cleanly or aborts
cleanly *before any state changes*. Throws after step 3 has started
fall through to the watchdog and are treated as programming errors,
not a recoverable lifecycle path.

## Wire model

Two envelope kinds carry engagement lifecycle (see
[response-envelope.md](./response-envelope.md) for the wire-side
authoritative shape):

- **Start** — `dispatch-response` envelope with
  `outcome.notes: [EngagementStartedNote]`. Pushed by the
  controller via `ctx.note(result.note)`.
- **Mid-engagement, complete, abort** — `activity-update`
  envelope, emitted out-of-band by the scheduler via
  `MessageApi.sendEnvelope`. There is no `CommandContext` in scope
  at those points.

Both envelope kinds fan out to all of an Avatar's Interactives via
`Avatar.handleEnvelope`; `frameId` is stamped per-Interactive at
the wire-delivery layer.

The sub-100ms `completed-sync` path is **wire-silent** — no
envelope, no notes. The engagement never appeared on the wire as
in-flight; the controller renders completion prose directly.

NPC engagements without `SensorMixin` are server-side observable
(shadows, audit) but never reach a wire. The scheduler's envelope
helpers gate on `MixinApi.isSensor(actor)` and skip the
`MessageApi.sendEnvelope` call when the actor lacks the mixin.

**Progress is derivable, not transmitted.** Don't send
`progress: 0.5` 50 times during a 5-second activity.
`EngagementStartedNote.startedAt + duration` is the only progress
contract; the client computes the rest from the wall clock. The
server only sends *event-shaped* mid-activity messages
(interruptions, detection, scheduled emissions with perceivable
side effects). No `engagement-progress` note kind exists.

**Completion mutations flow through state-sync, not the envelope.**
When `onComplete` commits (location change, item produced, property
mutation), those world-deltas ride on the **state-sync channel**,
not as additional envelope notes. The `engagement-completed` note
is a pure lifecycle signal — "this engagement reached its terminal
state successfully" — with no payload describing *what* changed.
Same for `engagement-cancelled`.

## Architectural decisions

The non-obvious calls — what could have been done differently and
why this shape was chosen.

1. **Engagement is a plain object, not a Stuff.** Activities are
   transient; they don't persist, don't get cloned, don't go
   through `StuffApi.create`. Plain class instances with a
   nanoid-minted `engagementId` are the right cost. The
   `Engagement` interface keeps Stuff out of the shape so a future
   isolate bridge could marshal engagements across without
   constructor entanglement.
2. **EngagedMixin storage is runtime-only.** Engagements don't
   persist — a server restart drops every in-flight engagement.
   Future `shutdown-lifecycle` subsystem will own the `restart`
   `AbortReason` augmentation; firing semantics are deferred. v1's
   short-running server makes this a non-issue.
3. **Lifecycle dispatch routes through `SchedulerRegistry`'s
   `activityRegistry`, not instance prototype.** Activities are plain
   TS classes (not Stuff-templated), so they don't pick up HMR like
   controllers do. The dispatch index is the smallest indirection
   that makes activity edits hot-reload-friendly: `start()` captures
   each engagement's own class under its type (capture-at-start — no
   registration step), and `SchedulerApi.reloadActivity(type)`
   re-points the entry to the freshly-reloaded class (as does the
   next `start` of that type); the next lifecycle fire uses the
   latest class's prototype. Construction-time `import` references
   would pin to the old class otherwise. **Caveat:** emission
   closures and per-instance field shape are pinned to
   construction; code-only edits to `onComplete` / `onAbort`
   bodies pick up cleanly, structural changes don't.
4. **100ms duration floor.** Below the floor the activity-shaped
   observability buys nothing the client can perceive (it's below
   network jitter); above it, the activity machinery pays back.
   Authors write `onComplete()` exactly once; the framework picks
   the schedule-or-run-now path. A 50ms `KneelActivity` gets
   correct synchronous semantics for free.
5. **Wire-silent `completed-sync`.** Sub-100ms activities never
   appeared on the wire as in-flight; emitting a
   `start → complete` pair would be churn for nothing. Controllers
   handle the rendering directly when they observe
   `status === 'completed-sync'`.
6. **`getHost?()` is the only eager-revalidation hook in v1.** The
   most-jarring failure mode (host destruction mid-activity) is
   cheap to detect; everything else falls to lazy revalidation at
   completion. Future per-activity opt-in to broader eager hooks
   lands additively.
7. **Engaged actors don't need `Mobile` or `Sensor`.** Composition
   is orthogonal — a forge-bound creature can be `Engaged` without
   being `Mobile`; an NPC running its own activities can be
   `Engaged` without being a `Sensor` (server-side observable, no
   wire). The scheduler narrows with `MixinApi.isSensor(actor)`
   before calling `MessageApi.sendEnvelope`.
8. **No separate non-checking `start()`.** A single method that
   does conflict resolution closes the lazy-caller footgun.
9. **Mid-traversal actor stays in source room (Wave 2+ note).** For
   eventual `TraverseActivity`: the actor's modeled position
   doesn't interpolate during a move. Perception of in-flight
   activity rides on the cadenced emission channel (footsteps,
   motion-trail), not on real-time position. Detection and
   interruption compose with everything we have without sub-room
   spatial granularity.

## The cast

| Element                              | Lives in                         | Role                                            |
|--------------------------------------|----------------------------------|-------------------------------------------------|
| `SchedulerApi`                       | `api/scheduler.ts`               | Thin facade over `SchedulerRegistry`            |
| `SchedulerRegistry`                  | `obj/SchedulerRegistry.ts`       | Active set, clock-handle timers, activity-class registry, lifecycle dispatch |
| `Engagement` / `DurativeActivity` / `SustainedEngagement` / `ScheduledEmission` / `StartResult` | `api/scheduler.ts` | Engagement interface family                     |
| `EngagedMixin` + `Engaged` + `EngagementSlot` + `ENGAGEMENT_SLOTS` | `lib/activity/Engaged.ts`    | Actor-side slot map + privileged mutators       |
| `AbortReasonRegistry` augmentation   | `lib/activity/Engaged.ts`        | Framework-intrinsic abort reasons               |
| `CancelController`                   | `obj/command/system/CancelController.ts`| `cancel` / `cancel <type>` verb logic           |
| `cancel.yaml`                        | `mud/cmd/system/cancel.yaml`            | Verb view                                       |

## Cross-references

- [response-envelope.md](./response-envelope.md) — the wire surface
  for engagement observability; `EngagementStartedNote`,
  `EngagementCompletedNote`, `EngagementCancelledNote`, and the
  `activity-update` envelope shape. The `AbortReasonRegistry`
  declaration-merging surface this subsystem augments lives in
  `@saxonberg/types` to keep the envelope's
  `engagement-cancelled.reason` typechecking across the package
  boundary.
- [hot-reload.md](./hot-reload.md) — `HotReloadApi.reload`
  semantics, the activity-class registry as the activity-subsystem
  HMR seam, the wizard-callable `SchedulerApi.reloadActivity`.
- [shell-alias.md](./shell-alias.md) — `AliasMixin` default-alias
  walk that picks up `EngagedMixin.defaultAliases` (`stop → cancel`).
- [call-security.md](./call-security.md) — the `ApiOnly` policy
  that gates `_setEngagement` / `_clearEngagement`; the
  `ExecutionContextApi.runRoot` pattern the scheduler uses to plant
  a synthetic SchedulerApi root frame for timer callbacks.
- [events.md](./events.md) — `EventApi.on(Events.StuffDestructed)`
  is what the host-destruction hook subscribes against.
- runtime-model.md — Node event loop,
  timing precision, multi-client reality; the cost model behind
  the 100ms floor.

## Future work

Wave 1 shipped the substrate; engaged consumers have since landed
(respiration, NPC behavior, the manual-build crafting verbs).
Forward-looking work — preserved as design reference in two
sibling slates:

- **Locomotion as activity** — see
  docs/slates/tails/locomotion-as-activity-slate.md.
  `TraverseActivity` for walk / climb / swim / fly / ride /
  sneak / crawl. Adds the `traverseSync` sibling to
  `Mobile.traverse`, a cached-destination getter on `Exit`, and
  migrates `Mobile.engagedMode` storage onto `EngagedMixin`. The
  physics-honest locomotion substrate (`speed` in m/s,
  `defaultDurationMs`, optional `Exit.durationOverrideMs`) lands
  with this wave. Deferred per current direction on game
  responsiveness — walking stays synchronous until durative
  content earns the slot.
- **Host-slot + non-locomotion activities** — see
  docs/slates/tails/host-slot-activities-slate.md.
  `MountActivity`, `DismountActivity`, `SitActivity`,
  `LieActivity`, `DriveActivity`, `ReadActivity`. Adds the
  `SlotApi` pending-claim extension (`claimPending` /
  `commitClaim` / `releaseClaim`) so cross-actor races on shared
  host slots (saddle, chair, driver seat) reject the loser
  cleanly rather than committing both. Deferred for the same
  reason.
- **Sound subsystem.** `SoundEvent` shape, `LocomotionMode.footstepCadenceMs`
  (real gait period), propagation walk, `noiseLevel` → dB SPL
  conversion. `TraverseActivity.emissions` becomes the first
  consumer when sound ships. The `ScheduledEmission` framework
  substrate is in place and unused in v1.
- **Bookmark-on-abort for `read`.** Per-book; a bookmark Stuff
  could carry the state. Defer until reading content earns it.
- **Per-activity opt-in to broader eager revalidation.** v1 ships
  only the host-destruction hook. Spellcasting-on-target wanting
  early-abort-on-target-death lands when a content case earns the
  framework addition.
- **Cross-activity coordinated commit** (lift-the-log-together) —
  multi-actor coordination slate territory.
- **Drain-and-reload wizard verb.** Composes `cancelByType` +
  `reloadActivity` per-actor — useful when an activity-class edit
  changes shape (new fields, new emissions, changed constructor
  invariants) such that in-flight engagements can't safely resume.
  Infrastructure ships in v1; the verb is a small future
  addition.
- **Server-restart lifecycle for in-flight engagements.** The
  `restart` `AbortReasonRegistry` augmentation lands when the
  shutdown-lifecycle subsystem ships.

## History

The Wave 1 build landed on the `activity` branch in three commits
between `435238f` and `d7319d0`. The build plan
(`docs/plans/activity-plan.md`) was kept in the tree at sweep time
as deferred-wave reference, then later extracted: Waves 2 and 3
became
docs/slates/tails/locomotion-as-activity-slate.md
and
docs/slates/tails/host-slot-activities-slate.md
— the proper artifact for open-ended design space — and the plan
was deleted. Git history preserves the original slate at
`docs/slates/activity-slate.md` and the original plan at
`docs/plans/activity-plan.md` for archaeology.
