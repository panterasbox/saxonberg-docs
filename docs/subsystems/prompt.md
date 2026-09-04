# Prompt substrate

Server-side substrate for interactive prompts — `choice` / `confirm`
/ `text` / `mqlObject` / `mqlMany` — plus the base-prompt rendering
pipeline the client's command-line prompt area depends on. Server
callers `await` a prompt; the substrate ships an envelope to the
client, the player responds, the await resolves.

The substrate is the second prompt-shaped wire channel sitting
alongside the dispatch-response / MQL-subscription pipelines.
Inbound dispatch lives in `Application.processUserMessage`; outbound
envelopes ride the same `onEnvelope` path as dispatch-
response and subscription deltas.

See:

- [docs/subsystems/response-envelope.md](./response-envelope.md) —
  the envelope family this substrate lives inside.
- [docs/subsystems/mql-subscription.md](./mql-subscription.md) —
  the pattern this substrate mirrors (per-Interactive substrate API,
  envelopes via `onEnvelope`, inbound dispatcher
  routes, disconnect cleanup).
- [docs/subsystems/command-spec.md](./command-spec.md) —
  cardinality vocabulary that consumes the substrate for
  disambiguation.

## File layout

| File | Role |
|---|---|
| `packages/server/src/mud/api/prompt.ts` | `PromptApi`, `PromptCancelledError`, `PromptValidator<T>`, `buildPromptContext`, `renderPromptRefresh` |
| `packages/types/src/index.ts` | Wire types (8 new Note kinds + 2 inbound messages) |
| `packages/content/platform/content/platform/cmd/system/prompt.yaml` + `platform/idea/cmd/system/PromptController.ts` | `prompt cancel` command |
| `packages/server/src/mud/lib/shell/Environment.ts` | `prompt.format` setting registration |
| `packages/server/src/mud/lib/command/CommandGiver.ts` | Refresh-Note injection at dispatch-response composition |
| `packages/server/src/backend/inbound/prompt.ts` | Inbound prompt routes (`handlePromptResponse` / `handlePromptCancel`), dispatched via `inboundHandlers` in `backend/inbound/index.ts` |
| `packages/server/src/backend/inbound/command.ts` | Empty-command short-circuit + `renderPromptRefresh` call (`:29-41`) |
| `packages/server/src/backend/Application.ts` | Disconnect cleanup (`handleUserDisconnect` triggers `interactive.teardownSubstrateState`, which runs `cancelPrompts`) |

## Surface

### `PromptApi`

```ts
choice(iact, label, choices, opts?): Promise<string>
confirm(iact, label, defaultAnswer?, opts?): Promise<boolean>
text(iact, label, opts?): Promise<string>
mqlObject(iact, label, matches: Stuff[], opts?): Promise<Stuff | null>
mqlMany(
  iact, label, matches: Stuff[],
  opts?: { min?: number; max?: number; foreground?; validate?; body? }
): Promise<Stuff[]>

cancel(promptId, reason?): boolean
cancelAll(interactive, reason): number

// Inbound entry points (called by Application.processUserMessage):
handleResponse(interactive, { promptId, response }): void
handleCancel(interactive, { promptId }): void
```

`opts`:

```ts
interface PromptOpts<T> {
  foreground?: boolean;  // default true; false joins stack without seizing input
  validate?: (response: T) => true | string | Promise<true | string>;
  body?: string | Mml;   // long-form prose for the terminal scroll
}
```

`mqlMany` additionally supports `min` / `max` bounds — the substrate
enforces them on response (under-min / over-max → emit
`prompt-validation-failed`, keep the prompt alive).

### Validator semantics

Async-permitted: `(response: T) => true | string | Promise<true | string>`.

- Returns `true` → await resolves with the typed response.
- Returns a string → substrate emits `prompt-validation-failed`,
  prompt stays alive, await is not settled. Player can re-respond.
- Throws / rejects → substrate emits validation-failed with the
  error message.

The validator sees **typed input**: `confirm`'s validator receives
`boolean`, `mqlObject`'s receives `Stuff | null`, `mqlMany`'s
receives `Stuff[]`. The substrate handles wire-string-to-T decoding
(including `mqlMany` bounds enforcement and `findById` lookups)
BEFORE running the validator.

Cancel-during-validate is race-safe: a prompt cancelled while a
validator is in flight has its `cancelled` flag set; when the
validator eventually settles, the substrate short-circuits without
calling `resolve`.

#### Why prompt validators allow async (and command validators don't)

Command validators are sync-by-design because the dispatcher's
validator pass runs in a deterministic sync block between MQL
resolution and controller execution. Async there would interleave
badly with the surrounding pipeline.

Prompts run in an already-async lifecycle: every push returns a
`Promise<T>`; the caller already wrote `await`. Async validation is
naturally absorbed by the prompt's "alive, awaiting next response"
state, and DB-uniqueness / external-service checks have a
non-awkward home.

The two surfaces stay separate; reach for `PromptApi`'s validator
when you have a prompt, for command validators when you're guarding
field shape.

### Two-channel inbound

```
Inbound message type        Routes to
─────────────────────       ────────────────────────────
command                     CommandGiver.executeCommand
mql-subscribe               MqlSubscriptionApi
mql-unsubscribe             MqlSubscriptionApi
prompt-response             interactive.handlePromptResponse  ← bypasses command bus
prompt-cancel               interactive.handlePromptCancel    ← bypasses command bus
```

`prompt-response` and `prompt-cancel` bypass the command bus
deliberately. Per-prompt responses are the prompt channel's
response shape; the command bus is for player-invoked actions.

Wholesale cancel rides the command bus separately via the `prompt
cancel` verb (next section).

### `prompt` command

```
prompt cancel    →  interactive.cancelPrompts('cancelled')
                    Reports the count of prompts cancelled.
```

The `prompt` namespace is reserved as the player's surface for
prompt-related actions. v1 ships one subcommand; future additions
(`prompt set <format>`, `prompt show`, etc.) land additively
against `PromptController.execute`'s switch.

Per-prompt cancel (the X button on a specific prompt) stays on the
wire as `prompt-cancel` — different semantics from the wholesale
verb form.

### Error class

```ts
class PromptCancelledError extends Error {
  readonly reason: 'cancelled' | 'host-disconnected';
}
```

Awaiting prompts reject with this on cancel. `cancelled` covers
both X-button cancellations and the `prompt cancel` verb;
`host-disconnected` is the disconnect-cleanup path.

Codebase precedent: `MqlPermissionError`, `SecurityError`,
`ContainmentError`. Use `if (err instanceof PromptCancelledError)`
to branch.

### Body MessageFrame correlation

Push with `opts.body` set emits a `shell.prompt` MessageFrame
BEFORE the `PromptEnvelope`. The frame's `payload` carries
`{ promptId }` so the client can correlate the long-form prose
with the prompt envelope (visual highlight, click-to-focus, etc.).

```ts
iact.promptMqlObject('Which sword?', matches, {
  body: Mml.compose`Multiple swords match: ${Mml.thing(rusty)} or ${Mml.thing(iron)}.`,
});
```

Push without `body` ships only the envelope — `label` is enough
context for short prompts; longer prose belongs in the terminal
scroll. The slate's snapshot-on-send pattern handles
response-side terminal echo client-side; the substrate emits
nothing on responses beyond the `prompt-dismissed` envelope.


### ⭐ `compose` opens on what is there

`ComposePromptOpts.initial` is the text the composer opens with — the
current body when the prompt is an **edit** rather than a creation. It
rides the `prompt-compose` note to a client that seeds the draft with
it.

> ⚠ Without it, "edit" silently means "retype": the box opens empty and
> whatever is posted replaces the whole body. `wiki edit` shipped that
> way and only a live drive found it — every test submitted `--body`,
> which skips the prompt path entirely.

A caller that has a current value should pass it. A caller creating
something new should not — there is nothing to open on, and an empty
box is the honest affordance.

## Lifecycle

### Push

1. Resolve the Interactive's holder via `getHolder()` and assert
   it's a Sensor — the prompt substrate's viewer check throws synchronously
   if the holder is missing or isn't Sensor-shaped. Push paths
   without a viewing surface (placeless Interactive, NPC-only
   Stuff) fail loudly rather than silently dropping prompts.
2. Generate `promptId` via `nanoid()`.
3. Store `ResolverRecord` in `#resolvers` (keyed by `promptId`)
   and add the id to `#byInteractive` (keyed by Interactive — for
   O(N) `cancelAll`).
4. If `opts.body` is set, fire the body MessageFrame.
5. Push the `PromptEnvelope` carrying the content Note.
6. Return the Promise.

### Response

1. `handleResponse(interactive, { promptId, response })` looks up
   the record.
2. Decode the wire response per kind:
   - `choice` / `text` — string passthrough.
   - `confirm` — `'yes'`/`'no'` → boolean.
   - `mqlObject` — `StuffApi.findById(stuffId)` → Stuff or null.
   - `mqlMany` — parse JSON array → validate bounds → map through
     `findById` → drop nulls.
3. If `validate` set, run it. On `true`, dismiss + resolve. On
   string, emit `prompt-validation-failed` and leave the prompt
   alive.
4. On no validator: dismiss + resolve immediately.

### Cancel

1. `handleCancel`, `cancel(promptId)`, or `cancelAll(interactive)`.
2. Set `record.cancelled = true` (race guard for in-flight
   validators).
3. Remove from `#resolvers` and `#byInteractive`.
4. Emit `prompt-dismissed` envelope with the reason.
5. Reject the await with `new PromptCancelledError(reason)`.

### Disconnect ordering

```ts
Application.handleUserDisconnect:
  1. Resolve Interactive from socketId.
  2. MqlSubscriptionApi.cancelAllForInteractive(interactive);
  3. interactive.cancelPrompts('host-disconnected');
  4. ConnectionManager.removeInteractive(socketId);
```

Both cancel passes run while the Interactive is still
addressable; pending awaits reject before the Interactive goes
away, so controllers' try/catch handlers can react cleanly.

### ⚠ A command sent while a prompt is open gets NO response

Measured on the wire during a live drive: with a `mqlObject` prompt
outstanding, an ordinary command from the same actor produces **no
dispatch-response, no note, no prose**. The client sends it, the socket
is healthy, and nothing comes back.

That is survivable when the prompt is answerable — but it is what turned
a focus bug into a session that appeared dead rather than one that
looped, and it cost an hour of diagnosis pointed at the wrong layer.
**A queued or refused command should say so.** Not fixed here; recorded
because the silence is the expensive part, not the queueing.

## Outbound delivery

Every server-pushed envelope (push / validation-failed / dismissed)
ships via `onEnvelope(holder, template)` — the same
Sensor pipeline the dispatch-response and subscription substrates
use. Shadow filters and audit observers consume prompt envelopes
on that channel.

The push envelope's `frameId` stamps from
`Interactive.nextFrameId` per existing ordering primitive, so
prompts order naturally relative to dispatch-response and
subscription deltas.

## Base-prompt rendering

The client's command-line prompt area (`here>`, `the brass kettle>`)
is server-rendered and shipped on every dispatch-response
envelope. MUD-style: every command refreshes the prompt; no live
subscription needed.

### Setting

```yaml
prompt.format: "{{ focus }}>"     # default
```

⚠ **`{{ focus }}` renders a focus ANCHOR as the thing, not as an id.**
After a disambiguation the binder anchors focus on `#<stuffId>` so
`$focus` re-resolves to what was picked rather than to the ambiguous
word (see [command-spec.md](./command-spec.md) § `updates_focus:`).
That fragment is MQL and the prompt is prose, so `buildPromptContext`
resolves a bare id anchor to its presentation — the prompt reads
`the house tablet>`, not `#wtYvAoxeDjtrMJku8NThs>`, which is what a live
drive got. Only the bare-anchor form: `here`, `here:bookcase:book` and
every other fragment are already the player's own words and show as
written; an id that no longer resolves falls back to the fragment.

Registered on `EnvironmentMixin` alongside other shell settings.
Players change via the existing `settings` / `set` vocabulary:

```
> set prompt.format "{{ focus }} ready>"
```

### Rendering pipeline

`renderPromptRefresh(giver)` reads the setting and renders via
`ProseApi.format` against the prompt Liquid context. v1 context
exposes one variable, `focus` (from `FocusedMixin.getFocus()` when
present, empty string otherwise). Future tokens (`posture`,
`location.name`, `time`) extend `buildPromptContext` additively.

Template render failures fall through to the bare default
(`{{ focus }}>`) so the player always sees a usable prompt.

### Wire injection

`CommandGiverMixin.executeCommand` appends the rendered
`PromptRefreshNote` to every `DispatchResponseEnvelope`'s
`outcome.notes`. The client updates its base-prompt state on
receipt.

### Empty-command short-circuit

An Enter-only command (`{ text: '' }`) bypasses the parser and
controller dispatch entirely. `handleCommand`
(`backend/inbound/command.ts:29-41`) ships a refresh-only
dispatch-response carrying just the `prompt-refresh` Note.
MUD-style "press Enter for a fresh prompt."

The short-circuit runs even when the avatar has no container — a
placeless avatar still sees their current prompt.

### Why MUD-style and not subscription

Live subscription on a complex format (5+ tokens with high-fire
sources like HP regen) would ship many redundant deltas per
second. MUD-style is O(1) per command. Live-status state
(HP/MV/combat ticker) belongs in dedicated widgets with their own
subscriptions — already in the Phase 3 cockpit plan. The prompt
is for short stateful labels (focus, mode, posture, time-of-day)
that don't fire often.

## The prompt strip — one slot, three occupants

In screen order: everything **waiting** sits above the input, the format
bar describes what the slot shows at rest, and the input itself holds
the **foreground** prompt when there is one.

```
WAITING  [CHOICE Tomas asks · true rim · 4m · background · waiting]
         [MQL-MANY Guild dues vote · guild vote 14 · 22m · background]
         one seizes input · the rest queue          [prompt cancel]
prompt   here>   ⋯
```

⭐ **Only one seizes input.** `foreground: false` joins the stack without
taking the slot, so a guild vote can arrive mid-forge and wait its turn
rather than interrupting. Picking one from the strip swaps it into the
slot; the rest keep waiting.

⚠ **A prompt remembers who asked.** Every prompt is pushed from inside a
running command, so the verb, its description and how long it has waited
are all knowable at push time — they ride the envelope. That is what
makes an abandoned prompt judgeable.

### ⚠⚠ Two cancels, and they are different acts

| Control | Verb | What dies |
|---|---|---|
| the `×` on a card | `prompt-cancel` | that one prompt |
| `prompt cancel` on the strip | `prompt cancel` | all of them |

**Cancelling is not dismissing a dialog.** It rejects the awaiting
command with `PromptCancelledError` — so the card has to say *which
command dies*: *"Cancelling abandons `true rim` — not just this card."*
A control that silently killed a running command would be the worst
kind of quiet.

⚠ **Failure is not dismissal.** A validator returning a string emits
`prompt-validation-failed` and the prompt stays alive. The UI must never
clear the answer.

## ⭐ A settled prompt wakes its card

`PromptLogic.cleanup` calls `MqlSubscriptionApi.notifyPromptSettled`.

`HOLD_WAKES_ON` records that an `unanswered` card needs no location
dependency because *the prompt's own resolution is what wakes it* — this
call is what makes that true. Before it, nothing poked the subscription
registry when a prompt resolved, so an `unanswered` card was
**immortal**: the player answered and the card stayed.

One known producer poking one known consumer, which is a method call
rather than a broadcast.

⚠ The regression test drives a **real** prompt rather than stubbing
`isPending`. A stub would have satisfied the assertion while the actual
wake path stayed broken — and a leaked spy between cases produces a
false pass of exactly the bug the file exists to catch.

## Cardinality vocabulary

The command-spec layer carries three optional knobs on `object` /
`objects` fields that route MQL resolution through the cardinality
matrix:

```yaml
fields:
  swords:
    type: objects
    mql: sword
    cardinality:
      min: 1
      max: 3
      # OR exactly: 3 (sugar for min == max)
    onExcess: take-all | prompt | truncate | error
    onShortage: error    # v1 only value
```

For `object` fields, cardinality is implicit `{ exactly: 1 }`;
`onExcess` accepts `top | prompt | error`.

The dispatcher applies `CommandApi.applyCardinalityPolicy` between
MQL resolution and controller execution. `applyCardinalityPolicy`
is async — when `onExcess: prompt` fires it pushes
`interactive.promptMqlObject` (object) or `promptMqlMany` (objects with bounds
from the cardinality spec) and awaits the player's pick. The
filtered Stuff list lands on the model unchanged shape; the
controller never sees the prompt round-trip.

The `prompt` policy degrades to an ambiguity error
(`controller-rejected { reason: 'ambiguous' | 'too-many' }`)
when no Interactive is attached to the context — a scripted /
NPC dispatch path has nobody to ask, so the policy falls back to
behave like `onExcess: error`. Player cancellation propagates as
`PromptCancelledError` from `applyCardinalityPolicy`; the
dispatcher (`CommandGiver._runChain`) catches and emits
`controller-rejected { reason: 'cancelled' | 'host-disconnected' }`.

See [docs/subsystems/command-spec.md](./command-spec.md) for
authoring patterns; [docs/subsystems/command-routing.md](./command-routing.md)
for the dispatcher decision matrix.

## What ships unused or deferred

- **Tier 2 / Tier 3 prompt kinds** (`numeric`, `multiChoice`,
  `password`, `paginated`, `quiz`). The slate defers per content
  demand.
- **Behavior flags** (`cancelable: false`, prompt timeouts,
  modality, multi-Interactive sync, preempting). No real use cases
  yet.
- **Token-format prompt richness beyond `{{ focus }}`.**
  Future tokens added as content demands.
- **Live MQL subscription for the base prompt.** MUD-style
  refresh was chosen for cost reasons; live subscription remains
  available if content demands it (per-token subscriptions through
  the existing MQL substrate, composed client-side).
- **FocusedMixin subscribableFields**. Not needed under MUD-style
  refresh.

## ⭐ A prompt card is PINNED, and settling closes it

`PromptLogic.cleanup` pokes `interactive.notifyPromptSettled`, which closes
the prompt's card with reason `answered`.

⚠⚠ **This is where the `unanswered` hold's guarantee went.** The card
substrate retired five hold conditions for one relevance window plus one
boolean — and `unanswered` was not symmetric with the other four. Its
subject was a pending **command**, and it is the one the design leans
on: *nothing that is still actionable ever leaves*. A prompt card that
timed out while still owing a reply is precisely the failure the hold
model was built to prevent.

So a prompt card opens **pinned**, which is what keeps the relevance
window away from it, and the settle is the only thing that ends it.
Same guarantee, one axis, no hold vocabulary.

⚠ The card carries **no body of its own** — only the `promptId`. The
client already holds one prompt model (the prompt queue) and the card
joins it by that id; rendering a second copy of the question would be
two renderings of one payload that can drift.

See [card-surface.md](./card-surface.md).
