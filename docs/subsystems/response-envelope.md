# Response envelope

The response envelope is the second wire channel server → client.
While `MessageFrame` carries prose (and lifelong server-side
broadcast), the envelope carries the **outcome of a dispatch attempt**:
a structured, machine-readable signal that says "your command was
parsed, validated, executed; here is what happened."

Two channels, two shapes, one ordering primitive:

```
                       ┌─── MessageFrame ────────┐
                       │  topic + body + payload │
                       │  prose / system logs    │
                       │  meta.frameId per Iact  │  ← one
                       └─────────────────────────┘   counter
                       ┌─── DispatchResponse ────┐    per
                       │  type + dispatchId      │   Inter-
                       │  outcome.status + notes │   active
                       │  frameId per Iact       │  ← spans
                       └─────────────────────────┘   both

       Avatar.handleEnvelope multiplexes to every connected Iact
       Iact.nextFrameId stamps both channels in monotonic order
```

`MessageFrame` continues to exist unchanged in shape (with one
addition: `meta.frameId?: number`). Prose, sensor-perspective
narration, log frames, and the input-echo all ride `MessageFrame`.
The envelope is a separate, complementary shape that the client
discriminates structurally (`frame.type === 'dispatch-response' &&
typeof frame.frameId === 'number'`).

## Wire shapes

All wire types live in `@saxonberg/types`:

```typescript
type Status = 'ok' | 'partial' | 'declined' | 'error';

interface DispatchOutcome {
  status: Status;
  notes: Note[];
}

interface DispatchResponseEnvelope {
  type: 'dispatch-response';
  frameId: number;        // per-Interactive monotonic; stamped at send-time
  dispatchId: string;     // = the originating commandId
  outcome: DispatchOutcome;
}

interface ActivityUpdateEnvelope {
  type: 'activity-update';
  frameId: number;
  engagementId: string;
  outcome: NoteOnlyOutcome;
}

interface PromptEnvelope {
  type: 'prompt';
  frameId: number;
  promptId: string;
  outcome: NoteOnlyOutcome;
}

type Envelope =
  | DispatchResponseEnvelope
  | ActivityUpdateEnvelope
  | PromptEnvelope;

// EnvelopeTemplate is the pre-stamping shape. Producers build this;
// the delivery layer stamps `frameId` per-Interactive.
type EnvelopeTemplate =
  | Omit<DispatchResponseEnvelope, 'frameId'>
  | Omit<ActivityUpdateEnvelope, 'frameId'>
  | Omit<PromptEnvelope, 'frameId'>;
```

`activity-update` ships in v1, produced by `SchedulerApi`'s
out-of-band lifecycle path (engagement completion, abort, replace).
The dispatch path stays the controller's `ctx.note(result.note)` for
the start case. See [activity.md](./activity.md) for the
producer-side framework.

`prompt` is **reserved** in v1 — defined in the union for stable
wire contract, but no producer ships it. The prompt-stack slate
plugs into it.

### Notes

Notes are the structured failure / partial-success / informational
signal in `outcome.notes`. They carry no prose — body text fires
through `Scene.send` on the parallel `MessageFrame` channel. The 16
kinds form one closed discriminated union:

**Glob / quantity** (emitted by `GlobbableApi.applyQuantity`,
forwarded by controllers via `ctx.note`):

- `quantity-clamped { field, requested, applied }` — lenient
  overflow; the helper clamped and acted on what was available.
- `quantity-clamped-rejected { field, requested, available }` —
  strict pre-check fail; nothing happened.
- `match-ambiguous { field, query, candidates: StuffRef[] }` —
  reserved; future MQL auto-disambiguation hook.
- `empty-result { field, query }` — candidate list was empty.
- `target-declined { target: StuffRef, reason }` — per-candidate
  decline from an action callback.

**Controller-side failure** (from the v1 controller audit):

- `controller-rejected { reason, detail? }` — generic, open-enum
  reason. Most-common failure shape.
- `mixin-missing { mixin }` — actor lacks a required capability mixin
  (`'WorkspaceMixin'`, `'BodyPlanMixin'`, `'ContainerMixin'`,
  `'VocalMixin'`, `'AliasMixin'`, `'EnvironmentMixin'`, `'MobileMixin'`).
- `locomotion-gate-failed { gate, mode }` — centralized in
  `LocomotionControllerBase.emitRejection`; the gate vocabulary is
  `'exit-mode' | 'posture' | 'body-plan' | 'enablement' | 'capability' | 'no-conveyance' | 'blocked' | 'door'`,
  mode is the short LocomotionMode name (`'walk'`, `'climb'`, …).
- `slot-occupied { host: StuffRef, slot, occupant?: StuffRef }` —
  required slot is taken (wear/wield/mount).

**Pre-controller dispatcher-emitted** (from `CommandGiverMixin._runChain` and `_executeOne`):

- `command-rejected { reason, detail? }` — pre-match failure;
  `reason` is one of `'parse-failed' | 'unknown-verb' | 'shape-fall-through' | 'bind-failed' | 'missing-subcommand'`.
- `mql-error { field, stage, detail }` — MQL threw during resolve;
  `stage` is `'desugar' | 'lex' | 'parse' | 'resolve'`.
- `validator-failed { field?, validator, detail }` — a field /
  verb / option / payload / subcommand validator returned a string.
  Validators MAY also emit their own richer notes (`controller-rejected
  { reason: 'cant-afford' }` etc.); the dispatcher's
  `validator-failed` rides alongside as the framework-tier fallback.
- `controller-error { controller, detail }` — exception thrown
  inside the controller (caught by the dispatcher's outer
  try/catch).

**Engagement lifecycle** (produced by `SchedulerApi` — see
[activity.md](./activity.md)):

- `engagement-started { engagementId, engagementType, startedAt, duration?, cancelable }`
  rides on the originating command's `dispatch-response` envelope.
  Controllers push it via `ctx.note(result.note)` on a
  `SchedulerApi.start()` outcome of `'started'` or `'replaced'`.
- `engagement-completed { engagementId }` rides on an
  out-of-band `activity-update` envelope, emitted by the scheduler
  when a `DurativeActivity`'s timer fires and `onComplete` runs
  cleanly. Mutation deltas flow through state-sync, NOT this note.
- `engagement-cancelled { engagementId, reason: AbortReason }` rides
  on an out-of-band `activity-update` envelope, emitted on every
  abort path (player cancel, replace, host destruction,
  precondition revalidation failure, `'thrown'` watchdog).
  `AbortReason = keyof AbortReasonRegistry`. The activity framework
  augments the registry with five framework-intrinsic reasons
  (`cancelled`, `replaced`, `preconditions-changed`,
  `host-destroyed`, `thrown`); other subsystems augment with their
  own reasons via their own modules.

The sub-100ms `completed-sync` path is **wire-silent** — no
envelope, no note. Controllers render completion prose directly
when they observe `SchedulerApi.start()` returning
`{ status: 'completed-sync' }`.

### Note kind versioning

Kinds are **append-only and per-kind-frozen**. A note's payload
shape is fixed once shipped — clients pattern-match by kind name and
rely on every payload field being present in the documented form. If
a kind's information shape needs to evolve, ship a new kind name
under which the new shape lives; the two coexist on the wire until
old producers retire. This keeps wire compatibility manageable
without a discriminator-version field on every note.

### Note ordering

The `notes` array is emission-ordered: a producer that emits notes
A then B has them at indices 0 and 1 respectively. No other
ordering guarantee — clients should treat the list as **semantically
unordered for rendering** and rely on the note kind + payload shape
to drive per-note behavior. Cross-note correlation (e.g., a
`quantity-clamped` and a `target-declined` from the same
`applyQuantity`) goes through payload fields, not through array
position.

### Auto-escalation

`outcome.status` follows the strongest-seen note unless a controller
pins it explicitly via `ctx.setStatus(...)`. The table lives
internally in `api/command.ts` (`autoEscalationFor`):

| Note kind                      | Implies status |
|--------------------------------|----------------|
| `quantity-clamped`             | `partial`      |
| `target-declined`              | `partial`      |
| `quantity-clamped-rejected`    | `declined`     |
| `empty-result`                 | `declined`     |
| `controller-rejected`          | `declined`     |
| `mixin-missing`                | `declined`     |
| `locomotion-gate-failed`       | `declined`     |
| `slot-occupied`                | `declined`     |
| `command-rejected`             | `declined`     |
| `mql-error`                    | `declined`     |
| `validator-failed`             | `declined`     |
| `controller-error`             | `error`        |

Rank ordering: `ok < partial < declined < error`. Strongest wins.
`match-ambiguous` and the three `engagement-*` kinds don't escalate
on their own; consumers branch on them as needed.

`setStatus` pins the status — subsequent notes still accumulate but
won't override the pinned value. Use it when a controller knows the
outcome shape ahead of time (rare; auto-escalation is usually right).

## The CommandContext accumulator

`CommandContext` is both the read-only data bundle controllers
inspect AND the accumulator they emit signals onto. The same
interface; the methods are the contract:

```typescript
interface CommandContext {
  // Data (populated by the dispatcher before execute()):
  commandGiver: Stuff & CommandGiver;
  interactive?: Interactive;
  location: Stuff & Container;
  commandText: string;
  executionId: string;
  commandId: string;
  verb: string;
  command: CommandDefinition;
  aliasExpansion?: AliasExpansionInfo;

  // Accumulator:
  note(n: Note): void;
  setStatus(s: Status): void;
  getNotes(): readonly Note[];
  getStatus(): Status;
}
```

Construction is `CommandApi.createCommandContext({ ... })`. The
factory's implementation hides the accumulator state behind a private
class; tests get the same factory as production code so synthetic
contexts behave identically.

Lifetime: **per `_executeOne` attempt**. `_runChain` mints a fresh
context for each assemble-stage match; only the claiming attempt's
context becomes the envelope. Shape-fall-through attempts are
discarded along with their (untouched) contexts. Pre-match failures
(unknown verb, parse error, bind error) report on the outer ctx.

## How a controller emits

The canonical pattern is **Scene.send + ctx.note** at every failure
site. The Scene frame gives the player something to read; the note
carries the structured signal. Both ride the same dispatch:

```typescript
class WearController extends CommandController<WearModel> {
  execute(model: WearModel, context: CommandContext): void {
    const giver = context.commandGiver;
    const target = model.target.stuff;
    if (!target) {
      MessageApi.scene(giver)
        .topic(MessageApi.Topics.sense.survey)
        .toSelf(Mml.compose`You don't have any '${model.target.raw}'.`)
        .send();
      context.note({
        kind: 'empty-result',
        field: 'target',
        query: model.target.raw,
      });
      return;
    }
    // … more failure gates …
    // success: just do the work; no return value.
    SlotApi.occupyAll(giver, target, [...slots]);
    MessageApi.scene(giver)
      .topic(MessageApi.Topics.sense.survey)
      .toSelf(Mml.compose`You put on ${Mml.thing(target)}.`)
      .toPeers(Mml.compose`${Mml.actor(giver)} puts on ${Mml.thing(target)}.`)
      .send();
  }
}
```

Controllers return `void`. The dispatcher reads the accumulator state
on `context` after `execute()` returns to assemble the envelope.
There is no `CommandResult`, no `success: boolean`, no `summary` —
those retired in Chunk 5 of the response-envelope build.

### Reasons are stable kebab-case

`controller-rejected.reason`, `target-declined.reason`,
`command-rejected.reason` are open-enum **kebab-case identifiers**.
Stable across releases. The detail string is human-prose; the reason
is machine-pattern-matchable.

The v1 reason vocabulary is whatever's grep-able in
`packages/server/src/mud/obj/command/` and the `LocomotionControllerBase`
gate names — `git grep "reason: '"` gives the inventory. New
controllers should reuse existing reasons where the semantics
match.

### `controller-error` is the dispatcher's catch — controllers don't try/catch

Wear / Wield / Mount used to wrap `SlotApi.occupyAll` in try/catch
and translate the thrown message into `summary`. They don't anymore.
The dispatcher's outer catch around `_executeOne` emits a uniform
`controller-error { controller, detail }` note for ANY thrown
exception. Controllers throwing is fine — it's the right shape for
programmatic-contract violations.

## Sensor pipeline

The envelope rides the Sensor pipeline, not direct-to-wire. This is
load-bearing for multi-device consistency, netdead Avatar reactions,
shadow-based interception (audit Sensors, debuffs), and NPC
programmatic self-observation. Triad on `SensorMixin`, parallel to the
MessageFrame triad:

```typescript
onEnvelope(envelope: EnvelopeTemplate): void {
  const transformed = this.filterEnvelope(envelope);
  if (transformed === null) return;
  this.handleEnvelope(transformed);
}

protected filterEnvelope(env: EnvelopeTemplate): EnvelopeTemplate | null {
  return env;     // shadowable
}

protected handleEnvelope(_env: EnvelopeTemplate): void {
  // subclass override; Avatar fans out, NPCs default no-op
}
```

`Avatar.handleEnvelope` is the multiplexing fan-out:

```typescript
protected override handleEnvelope(env: EnvelopeTemplate): void {
  const app = Avatar.getApplicationInstance();
  for (const interactive of this.interactives) {
    app.sendEnvelopeToInteractive(interactive, env);
  }
}
```

A netdead Avatar (no connected Interactives) hits the for-each as a
no-op — but the `handleEnvelope` body itself ran, so any server-side
reactions (shadow filters, audit observers, NPC AI that watches an
Avatar) fire regardless of wire state. Symmetric to today's
`Avatar.handleMessage`.

`MessageApi.sendEnvelope(recipient, template)` is the lone delivery
chokepoint. Producers (only the dispatcher in v1) call this; nothing
else calls `sensor.onEnvelope` directly.

## frameId — the shared ordering primitive

Both the envelope and `MessageFrame` get a `frameId: number` stamped
per-Interactive at the wire-delivery layer. The counter lives on
`Interactive` as a hard-private `#frameCounter` exposed through
`nextFrameId()`. It starts at 0, increments to 1 on the first
`nextFrameId()` call, and **resets on reconnect** (a fresh
`Interactive` instance has a fresh counter).

```
Avatar with 2 Interactives, A and B
  Scene.send → MessageApi.sendMessage → Avatar.handleMessage
    → for each Iact: Application.sendMessageToInteractive
      → stamps meta.frameId from this Iact's counter
      → Backend.sendMessageToSocket

  MessageApi.sendEnvelope → Avatar.handleEnvelope
    → for each Iact: Application.sendEnvelopeToInteractive
      → stamps frameId from this Iact's counter
      → Backend.sendEnvelopeToSocket
```

Same logical frame multiplexed to two Interactives gets two
different `frameId`s — each Interactive's wire stream is monotonic
from its own perspective. Both channels share the counter so the
client sees gap-free monotonic ordering across all server → client
traffic on a single Interactive. State-sync (when it ships) reads
the same counter — single ordering primitive, three channels.

`MessageFrame.meta.frameId` is **absent at compose-time** (Scene.send
and MudlogApi don't set it) and populated at the
`Application.sendMessageToInteractive` send-time chokepoint. Producers
ignore it; consumers (the client) use it for gap detection.

## Pre-controller failure paths

The dispatcher emits structured notes for every failure path before
the controller's `execute()` runs. The five `command-rejected`
reasons:

| Reason                | Emitted at                               | Outer or attempt ctx |
|-----------------------|------------------------------------------|----------------------|
| `parse-failed`        | `executeCommand` parse error branch       | outer                |
| `unknown-verb`        | `_runChain` empty match list             | outer                |
| `shape-fall-through`  | `_runChain` after every match's `assemble` returned a shape error | outer                |
| `bind-failed`         | `_runChain` on assemble's bind error     | outer                |
| `missing-subcommand`  | `_executeOne` when subcommand was needed but absent | per-attempt          |

Plus two pre-execute kinds from the resolve / validate pipeline
(`resolveModel` for MQL, `runValidators` for sync validator checks,
with `preloadValidatorDeps` awaited in between):

- `mql-error { field, stage, detail }` — wraps MQL resolve calls; on
  throw emits the note and short-circuits the dispatch.
- `validator-failed { field?, validator, detail }` — every time a
  validator returns a string. Categorical `validator` label is the
  scope ('verb' / 'field' / 'option' / 'payload' / 'subcommand:X').

Validators MAY also call `ctx.note(...)` with richer kinds before
returning the string — the dispatcher's `validator-failed` rides
alongside as the framework-tier fallback. Generic validators
(`mustBeContainable`, `canReach`, `requiresAnimate`) stay
note-silent; specialized domain validators (`canAfford`,
`notOnCooldown`) opt into the richer signal.

Controller exceptions land as `controller-error { controller, detail }`.

### Envelope vs scene split

The dispatch-response envelope is the **machine channel**:
typed `Note` records for bots, scripts, replay, automated
testing. The Scene composer is the **human channel**: prose,
Mml, topic-routed prose-render frames for player clients.
Framework-emitted failures (parse / MQL / validator /
controller-throw) put the structured note on the envelope AND
fire a prose scene on the topic `shell.error` so a
player typing a bad command sees WHY without the client needing
to render envelopes.

The sweep is dispatcher-driven: at the end of `executeCommand`,
the dispatcher walks `claimingCtx.getNotes()` and for each note
calls a module-private `proseForFrameworkNote(note)` helper (in
`lib/command/CommandGiver.ts`) to map it to its player-facing
prose. Auto-prosed kinds today are `command-rejected`,
`mql-error`, `validator-failed`, and `controller-error` —
exactly the kinds the dispatcher / validator / MQL framework
emit without a controller getting a chance to fire prose itself.
Controller-side notes (anything a controller puts on `ctx` after
its `execute()` started running — `controller-rejected` for
domain refusals, `mixin-missing`, `locomotion-gate-failed`,
etc.) return `null` from `proseForFrameworkNote` and are NOT
auto-prosed: controllers are expected to fire their own scenes
with domain-specific wording at the same site they note.

(Prose phrasing lives module-private with the dispatcher, not on
an Api class. Presentation logic doesn't belong on `mud/api/`
surfaces — the historical exception, the static `DescribeApi`, was
retired in favor of `Stuff.getPresentation()` (self-presentation
lives on the host), and the client takes over more of the rendering
over time.)

The `shell.error` topic is defined under
`TOPICS.shell.error`. Add it to the client's `renderTopics`
list to surface framework prose in the terminal.

## Input echo — `shell.diagnostic.{info|warn}`

The dispatcher emits an **input echo** MessageFrame at start of
dispatch, regardless of parse outcome. Topic
`shell.diagnostic.info` (parseable) or `shell.diagnostic.warn`
(parse failed). Payload:

```typescript
{
  kind: 'issued';
  rawText: string;
  expandedText?: string;     // present when alias expansion fired
  verb?: string;
  parseError?: string;       // present iff parse failed
  dispatchId: string;        // = the originating commandId
  originInteractiveId?: string;  // the Interactive that issued
                                  // (absent for programmatic dispatch)
}
```

Use cases:

- **Multi-device echo**: an Avatar's other connected Interactives see
  what their sibling typed. The client filters out its own echo by
  comparing `payload.originInteractiveId` against its stashed
  `selfInteractiveId` (carried on `session.link`'s
  payload as `interactiveStuffId`).
- **Audit trail**: server-side audit Sensors observing log frames see
  player input independent of any dispatch outcome.
- **Replay capture**: structured `kind: 'issued'` records replay
  cleanly.
- **Dev debug**: terminal panel can render input as it's processed
  vs. after the response envelope lands.

The echo fires **once per `executeCommand`** call. The topic
namespace `shell.diagnostic.*` is repurposed from the retired
auto-emit (see "Retired things" below).

## What got retired

The pre-envelope dispatch had a different shape. The retirements:

### `CommandResult`, `success`, `summary`, `pass`

Gone. Controllers' `execute()` returns `void`. The dispatcher reads
outcome state from the `CommandContext` accumulator, not from a
return value.

`pass: true` (defer to the next match in the chain) retired alongside.
Same-verb collisions resolve at the assemble stage (shape vs bind), or
via dynamic contributions on the recency stack — see
[command-routing.md § Dynamic contributions](./command-routing.md).

### Auto-emit MudlogApi command-outcome frame

`CommandGiverMixin.executeCommand` used to fire one MessageFrame per
command at `shell.diagnostic.{info|warn}` with body `<verb>: <tail>`
and payload `{verb, success, commandText, executionId}`. Gone.
Replaced by:

- **Dispatch-response envelope** — structured outcome.
- **Input echo** — `kind: 'issued'` payload at the same topic, fired
  at start of dispatch instead of end.

The topic namespace `shell.diagnostic.*` survives; the producer and
shape changed.

### `DispatchApi`

The Chunk 2 plan introduced a `DispatchApi` for the auto-escalation
table and rank lookup. It folded into `api/command.ts` as
module-private helpers (`autoEscalationFor`, `STATUS_RANK`) — the
only consumer was `CommandContextImpl.note`, so an Api class was
unnecessary indirection.

### Glob's parallel note types

`glob.ts` used to define its own `QuantityClampedNote`,
`QuantityClampedRejectedNote`, `EmptyResultNote`, `TargetDeclinedNote`
interfaces with subtly different shapes (no `field`, `target: Stuff`).
Now glob emits the canonical `@saxonberg/types` shapes directly —
`applyQuantity` takes a `{ field, query? }` opts so it can stamp
`field` per the canonical contract; `target-declined.target` is
constructed via `MessageApi.refOf(candidate)` for wire safety.

## Wire correlation

Same `dispatchId` (= the originating `commandId`) appears on:

- `DispatchResponseEnvelope.dispatchId`
- Input-echo `MessageFrame.payload.dispatchId`
- Prose `MessageFrame.meta.commandId` for every Scene fired during
  the synchronous span of the command
- `MessageFrame.meta.causingCommandId` on async-aftermath frames
  propagated via `ScheduleApi`

The client correlates by `dispatchId` to group all traffic from a
single dispatch into one logical bundle. `frameId` orders frames
within a single Interactive's stream; `dispatchId` groups across
streams.

## Frame ordering across channels

v1 emits the controller's Scene frames during `execute()`, then the
dispatch-response envelope at the end of `_executeOne`. Same
WebSocket send order per-Interactive.

If a future async controller sends Scenes after returning (via
`ScheduleApi`), the envelope may arrive **before** some peer Scene
frames on other clients. But on the **actor's** connection, the
actor's Scene frames fire before the envelope is built — ordering
is intact on the dispatch's own Interactive.

The async-aftermath frames carry `causingCommandId` (not
`commandId`), so the client can recognize them as "this happened
because of my command, but my command already responded."

## Architectural decisions (preserved from the retired slate)

These were settled during slate design and are load-bearing for the
shipped subsystem:

1. **Envelope is a NEW wire frame shape, alongside `MessageFrame`** —
   not a superset, wrapper, or replacement. Two channels, two shapes,
   correlated by id. Existing prose flow untouched.
2. **Envelope carries no prose.** No `scene` field. Prose stays on
   per-emit `MessageFrame`s. Outcome is dispatch-scoped; prose is
   per-emit; different lifetimes shouldn't share a frame shape.
3. **`MessageFrame.topic` survives unchanged.** Envelope `type` is an
   orthogonal wire-interaction axis. Topics answer "what kind of
   message"; envelope types answer "what kind of wire interaction".
4. **`frameId` is stamped at send-time per-Interactive**, not at
   producer-time. Multiplexing: one Avatar can have N Interactives;
   each Interactive's wire stream must be monotonic from its own
   perspective.
5. **`MessageFrame.meta.frameId` is optional but always populated by
   `Application.sendMessageToInteractive`.** Closes gap-detection
   coverage on the prose channel. Producers leave it absent.
6. **Envelope flows through the actor's Sensor pipeline, not
   direct-to-wire.** Mirrors the `MessageFrame` pipeline exactly so
   shadow-based interception, audit Sensors, NPC self-observation
   etc. work identically across channels.
7. **`dispatchId` = the originating `commandId`.** Same id, two
   surfaces.
8. **`witness` envelope frame type is NOT shipped in v1.** Peer-side
   wire output is prose-only via `MessageFrame`s with
   `audience:witness` tag. Additive when a structured peer-event
   signal earns it (NPC reactions, audit Sensors, LLM-agent
   observers).
9. **`engagement-cancelled.reason` is type-extended via the activity
   framework's `AbortReasonRegistry` declaration-merging surface.**
   Same open-enum-with-compile-time-extension pattern as
   `MqlMatchVia`. The empty interface lives in `@saxonberg/types`;
   the activity framework augments with five framework-intrinsic
   reasons. See [activity.md § The Abort taxonomy](./activity.md).

## Cross-references

- [messaging.md](./messaging.md) — `MessageFrame` shape, Scene
  composer, MudlogApi, topic / tag taxonomy, the `Sensor` pipeline
  for prose
- [command-routing.md](./command-routing.md) — dispatch shape: verb
  matching, recency stack, `CommandContext` accumulator API, validator
  conventions, dynamic contributions (the retired `pass: true`
  replacement)
- [glob.md](./glob.md) — `applyQuantity` opts shape (`{ field, query? }`)
  and the canonical-shaped notes the helper emits
- [locomotion.md](./locomotion.md) — `LocomotionControllerBase.emitRejection`
  and the `locomotion-gate-failed` note's gate vocabulary
- [slot.md](./slot.md) — `slot-occupied` note kind, wear / wield /
  mount audit
- [connection.md](./connection.md) — `Interactive` lifecycle (the
  `nextFrameId` counter lives here; resets on reconnect),
  `ConnectionEstablishedPayload.interactiveStuffId`
- [activity.md](./activity.md) — `Engagement` framework; the
  producer behind `activity-update` envelopes and the
  `AbortReasonRegistry` augmentations
- `docs/slates/tails/mql-subscription-slate.md` — the live-state
  substrate (formerly designed as state-sync); will share the
  `Interactive.nextFrameId` counter for its own ordering primitive

## History

The slate (requirements), build plan, and per-controller audit
manifest were deleted once the build shipped — their load-bearing
content is in this doc and the architectural-decisions section
above. Git history preserves them for archaeology:

```
git log --oneline a94be73^..ad984ed -- docs/slates/response-envelope-slate.md
git log --oneline a94be73^..ad984ed -- docs/plans/response-envelope-plan.md
git log --oneline a94be73^..ad984ed -- docs/plans/response-envelope-audit-manifest.md
```

The build itself rolled out in 13 commits on the `response` branch
between a94be73 and ad984ed; `git log` gives the chunk-by-chunk
narrative.
