# Reactions — act-scoped emote aggregation

The reactions subsystem turns "everyone reacting to one thing" from N
diegetic lines into one batched, attributed counter — without forking
the emote path. This doc is the source of truth for the area; read it
before editing.

## The model

**A reaction is an ordinary emote carrying one extra scope:
`inReactionTo: <commandId>`.** Everything that makes a reaction *feel*
different from a wave — the tally, the toggle, the threshold
aggregation, the renown event — is a thin layer (`ReactionRegistry`)
hung off the emote send. There is **no parallel dispatch**.
`EmoteOptions` gained exactly one optional field, and
`SoulMixin.emote`/`emoteFree` gained one "if scoped, poke the registry"
hook.

The act being reacted to is keyed by **`meta.commandId`** — the nanoid
minted once per dispatch in `CommandGiver` and stamped onto every frame
by `Scene.send`. The **subject** (credited author) of a reaction is the
act's speaker. Two witnesses who saw *different* rendered names for the
same speaker (per-viewer late-bound `Mml` naming) both react → one
aggregate on the shared `commandId`, crediting the one true author. This
cross-viewer aggregation falls out **for free** from keying on
`commandId` + capturing the subject at the producer send site.

### Reactable act-kinds (closed v1 set)

`ReactionApi.REACTABLE_TOPICS` / `ReactionApi.isReactableTopic`:

- `speech.vocal` — say / shout / whisper, which the S2 topic collapse
  folded onto one name. ⚠ This list read `speech.vocal, speech.quiet`
  before Wave 6; `speech.quiet` is not in the set and the literal itself
  carried three copies of `speech.vocal`, left over from the same
  collapse. `reactableTopics.test.ts` now asserts the **count**, because
  a membership assertion passes either way.
- `act.emote`
- `speech.channel`
- `act.combat` — a **dramatic** combat beat (a hit / break /
  down / kill; tick/pressed stay silent). The producer is `CombatNarration`,
  which mints its own `commandId` and calls `noteReactableAct` at the
  narration site (the beat runs in a detached scheduler root). See
  [combat.md](./combat.md).

A frame is reactable iff **topic ∈ REACTABLE_TOPICS ∧ `commandId`
present ∧ a broadcast audience exists**. A whisper is reactable exactly
like say/shout — it fans a peers frame to in-range overhearers who
share the one `commandId`; the lower `acousticDb` shortens reach, it
does not make the act private.

**A reaction's own frame is never noted reactable** (the gate in
`SoulMixin` on `inReactionTo === undefined`). That stops the regress.

## The one behavioral divergence: volume-gated fan-out suppression

Below the threshold a scoped emote behaves **identically to a normal
emote**: it fans out as a real diegetic line (`"Iffy nods at Vera's
words"`) *and* is tallied. **At/above the threshold the per-emote
fan-out is suppressed** — the emote's `Scene.send()` is skipped — and
the reaction feeds only the batched counter. That is the *entire*
difference. `ReactionRegistry.onScopedEmote` returns
`{ suppressFanOut }` **synchronously** (it reads in-memory counts), and
the mixin returns early before `scene.send()` when suppressing.

`aggregated` is computed *including* the current reaction and is
**sticky** — the threshold-crossing reaction suppresses its own line
(no flicker), and once a phenomenon it stays in counter mode.

## The aggregation contract — server numbers, client drama

`ReactionRegistry.flush()` runs on a **fixed-cadence timer**
(`ScheduleApi.recurring`, `fixed-rate`, default 200 ms, clamped to
[150, 250]) — **NOT** `setImmediate`. The mql-subscription substrate
uses `setImmediate` to collapse *one synchronous burst*; reactions
instead need a *wall-clock window* that collapses an unbounded stream of
independent command dispatches (1000 reactions/sec, each its own
dispatch, never in one tick) into ~5–10 flushes/sec **regardless of
throughput**. Per-tick wire cost is a function of *audience × cadence*,
not reaction count. This is the architectural crux — the scale-bound
test exists to lock it in.

On each tick, for each recipient sink the registry packs every act that
*moved in that sink's view* into one `ReactionDeltaEnvelope` carrying
**tag-grouped absolute counts** (the bounded backbone) **plus a small
capped familiar-biased attributed sample** selected *per-recipient* via
contacts/recognition. The full reactor-set is **pull-only on expand**.

Counts are **absolute, not deltas**. The client *replaces* its bucket
counts on receipt and synthesizes animation from the change; it never
sums.

## The registry

`obj/ReactionRegistry.ts` — singleton `Idea` at `/obj/ReactionRegistry`,
in-memory authority, **nothing persisted** (the Sybil-gameable trap the
reputation build avoids). State uses `private`, not `#private` (the
call-security-proxy rule). Every public method is gated
`FromModule('/api/reaction#ReactionApi')` + `SelfOnly`; the thin
`api/reaction.ts` `ReactionApi` facade is the only legitimate caller.
Mirrors the `MqlSubscriptionApi ↔ MqlSubscriptionRegistry` split, minus
the HMR logic-singleton indirection (registry state survives an Api
reload because it lives on the Stuff).

Key state: `acts: Map<commandId, ActRecord>` (`{ subjectId, scope,
createdAt, reactions: Map<reactorId, ReactorReaction>, aggregated }`),
`dirty: Set<commandId>`, `scopeSinks`, `interactiveSinks`,
`lastBySubject` (for `--to`), the per-Interactive gutter ring
(`frameId → commandId`, for `--msg`), and the flush `ScheduleHandle`.

- **`onScopedEmote`** (sync): **add-only / idempotent** tally on
  `(reactorId, emote)` — a new reaction flips on and fires
  `ReactionFiredEvent`; re-firing the same emote is a no-op tally-wise
  (no renown re-fire, though the diegetic line still fans out below
  threshold). Reacting never toggles off. Recomputes `aggregated`,
  returns `{ suppressFanOut }`. **Glyph-gated:** an emote with no emoji
  (and free-form text) is *not tallied* — it returns immediately and
  just renders as its diegetic line. Reactions aggregate by glyph for
  now, so the chip rail stays emoji-only and `total` equals the sum of
  the visible chips.
- **`removeReaction`** (sync): the explicit un-react — drops the
  reactor's `(reactor, emote)` reaction. No renown, no diegetic line;
  the count just falls. This is the *only* way to decrement a reaction,
  so it's GUI-driven (clicking an active chip) and exposed on the CLI as
  `react --remove`. The split exists because toggle-on-re-react
  surprised; an explicit removal is unambiguous.
- **`noteReactableAct`** (idempotent on `commandId`): captures
  `subjectId` + `scope` the first time a producer composes a reactable
  frame. **Never keys on `causingCommandId`** (that fans one cause into
  many acts).
- **`flush`**: the fixed-cadence sink-agnostic broadcaster (above).
- **`handleExpand`**: the full recognition-named reactor set on demand.

### Scopes

`location:<containerId>` (the co-present circle) or
`channel:<groupRef>` (the channel audience).
`ReactionApi.locationScopeFor(stuff)` computes the in-room scope
(Containable → environment; pure-Container → self).

A normal player's `InteractiveReactionSink` sees a `location:` scope by
**current container match** (handles walk-in/walk-out for free) and a
`channel:` scope once the holder has engaged it. The read-only
`service:broadcast` principal — which has **no Interactive** — is
reached not by a sink but by the **`ReactionScopeDeltaEvent`** the flush
fires per moved scope; `BroadcastFeed` taps it the same way it taps
`Events.StreamStateChanged`. This keeps the registry sink-agnostic.

### GC

A single TTL pass inside `flush()` drops acts older than the TTL
(default 5 min), approximating the chat ring's 200-cap conceptual
bound. Room speech/emotes have no persistent ring, so a TTL is the
correct GC for them. No per-ring eviction listener in v1.

## The `react` verb

`cmd/social/react.yaml` + `obj/command/social/ReactController.ts`,
contributed via `SoulMixin.commandContributions.self` (reacting
dispatches an emote, so it requires Soul).

```
react [--to <person>] [--msg <#>] [--remove] <emote-expression>
```

- selector-less (`re ;smile`) → the most recent act delivered in view
  (the **frictionless primary path** — low friction is load-bearing for
  adoption);
- `--to <person>` (parser-typed `object`) → that subject's most-recent
  reactable act (`lastReactableActBy`);
- `--msg <#>` → a specific gutter number → `commandId` via the
  per-Interactive ring (`resolveGutter`);
- `--remove` → un-react: drop your `<emote-expression>` reaction from
  the act instead of adding it. Routes to `ReactionApi.removeReaction`
  (NOT the emote path — no diegetic line). Mostly GUI-driven; rarely
  typed.

The act-selector is parser-typed — the controller never string-sniffs.
The emote-expression is the sole greedy positional; on an **add** it is
dispatched opaquely through the existing emote path (`SoulApi.resolve` →
`EmoteGrammarRunner.bind`, or `emoteFree` on a free-form miss) with
`inReactionTo` set, so the controller does **not** touch the registry to
mutate tallies. On `--remove` the controller resolves the canonical verb
and calls `removeReaction` directly. The `re` alias completes the cheap
surface.

### Gutter → commandId (server-side)

The gutter number the client shows is the per-Interactive `meta.frameId`
(`Interactive.nextFrameId`). The durable act key is `meta.commandId`.
`Application.sendMessageToInteractive` records `(frameId → commandId)`
into the registry's bounded ring whenever a reactable-act frame is
delivered (and registers the player's sink). `react --msg <n>` resolves
through that ring; a client-UI click emits the same gutter number.
**Input never carries a client-supplied `commandId`.**

## The renown event + the reception seam

`lib/events/ReactionFiredEvent.ts` — fired on a **flip-on** only,
carrying raw, **uninterpreted** emote + tags (no valence/polarity, no
score), `scope`, and a `selfReaction` flag. **The renown build is now its
consumer** (the aggregator the event was shaped for) — `RenownLogic` taps
it into the durable `renown_events` log; `ReactionRegistry` itself stays
ephemeral. See [renown.md](./renown.md).

Renown's *second* input — passive **reception** ("being heard") — also
hangs off the reactable-act machinery, on the **receive** side. The
producer-site `noteReactableAct` (Vocal/Soul/ChannelCatalogue) records the
act's `subjectId` + `scope`, exposed by `ReactionApi.actInfo(commandId)`;
`SensorMixin.onMessage` fires `CommReceivedEvent` for genuinely-perceived
communication frames (topic-gated via `MessageApi.isCommunicative`), and
renown recovers the speaker from `actInfo`. Distinct from a reaction (an
*active* signed signal) — reception is *passive*, small, and
log-saturated.

## Wire envelopes (`@saxonberg/types`)

- `ReactionDeltaEnvelope` (`reaction-delta`) — fixed-cadence, one per
  recipient per tick; `acts: ReactionActState[]` (each
  `{ commandId, subjectId, scope, buckets, sample, total, aggregated }`,
  counts absolute).
- `ReactionExpandMessage` (`reaction-expand`) inbound /
  `ReactionExpandResultEnvelope` (`reaction-expand-result`) — the full
  reactor set on demand.
- `MessageFrame.meta.inReactionTo` — stamped on a reaction's own frame
  for client render-correlation.

## Client

`store/index.ts` `reactions: Record<commandId, ReactionActState>` slice
+ `applyReactionDelta` (**replace, never sum**) /
`applyReactionExpandResult`; `store/reactionActions.ts` wires the
envelope handlers + the `reaction-expand` pull. The `Frame` carries
`commandId` / `inReactionTo` / `frameId` (the gutter) for
render-correlation + the per-row react selector.

The **`ReactionBar`** component (`components/ReactionBar.tsx`) renders
**inline at the end of the message body** (not a dedicated row): the
emoji chips trail the text only when reactions exist, and the react
**`+`** is hover-revealed (visually-hidden-but-focusable at rest — zero
footprint, but keyboard-reachable via `:focus-within`; the shared
`REVEAL_REACTION_ADD` block reveals it on `FrameRow:hover`, focus, or
open palette).

Each act-state's **`mine`** field (the recipient's own reactions) drives
the chip command: an active chip (`mine` contains its emote) sends
`react --remove --msg <#> ;<verb>` (un-react), an inactive chip or the
quick-react palette sends `react --msg <#> ;<verb>` (add). Each
bucket's **`reactors`** (viewer-named, present only while the bucket is
small enough — ≤ `NAME_LIST_CAP`) powers the **"who reacted" hover** on
the chip; large buckets show just the count. Crucially, every reaction
affordance is a **clickable command** — it routes through the shared
`onCommandClick` / `onCommandPreview` handlers (preview the command in
the command bar on hover, send on click) exactly like every other
clickable in the client; it never sends a websocket frame directly.

### The emote picker (Wave 6)

⭐ **There is no reaction picker, because a reaction is an ordinary emote
carrying `inReactionTo`.** What the `+` opens is an *emote* picker that
happens to be pointed at an earlier act — which is the same thing the
data model says.

The quick row is six emoji-bearing emotes; `all` opens the full
slot-aware palette (`components/social/EmotePicker.tsx`), shared verbatim
by the desktop popover and the phone sheet so **grammar survives the
phone**. Every cell prints its canonical verb under the glyph: emoji are
an alias surface over verbs, and this is the most-used interaction in the
product — the last place the command line should go quiet.

**The palette is DERIVED, never listed.** It draws from
`ConnectionEstablishedPayload.emoteCatalogue` — a session-cached
projection of `SoulCatalogue`, canonical verbs only, riding the same
payload and the same semantics as `topicCatalogue`. It replaced a
hardcoded six-entry `{ verb, emoji }` array in `ReactionBar`;
`noHardcodedEmoji.test.ts` guards both files against it returning.

**Slots.** An emote with grammar (`;wave <at> <manner>`) cannot fit a
flat grid, so selecting one reveals a control per declared slot in
**declaration order** — the order `EmoteGrammarRunner.bind` consumes
tokens in. `stuff` slots render solid (MQL-resolved server-side), `free`
slots dashed. A blank `stuff` slot is skipped (the binder skips on
mismatch, so `;wave happily` still binds); a blank `free` slot stops the
command rather than shifting a later value into an earlier slot.

**One `composeReactCommand`** feeds the preview and the send, and
`reactCommand.test.ts` asserts their *equality* rather than writing the
expected string twice. The selector is always explicit `--msg <gutter#>`:
the selector-less form means *the most recent act in view*, which is not
what a picker opened on message 112 does.

⚠⚠ **The verb goes in BARE — `react --msg 22 nod`, never `;nod`.** `;`
marks an emote only at the START of a line (`msh.detectEmotePrefix`
checks the first character); mid-line it separates statements. So
`react --msg 22 ;wave` parsed as `react --msg 22` — an arity failure,
because `expression` is required — followed by `wave`, and produced:

```
That doesn't match any known command shape: react.
I don't understand 'wave'.
```

**The shipped client had composed exactly that for every chip and every
quick-react, so reacting from the GUI had never worked.** It was found by
driving; no test could see it, because every test asserted the client's
own string against itself. The verb's help already said the expression is
"any ordinary emote, exactly as you'd type it bare".

⭐⭐ **The sheet sends DIRECTLY, and it is the one exception to the
command sheet.** `CommandSheet` exists to insert a naming moment a phone
otherwise lacks — every affordance gets one, deliberately, with no
obvious-case exception. **The emote sheet is already that moment**: it
prints the verbatim command above its own send control. Routing it
through the command sheet added no pedagogical dividend and asked the
same question twice — found by driving, where reacting on a phone cost a
hold plus three taps and showed `react --msg 22 agree` on two
consecutive sheets. `LayoutProps.onCommandSend` is the direct path;
`EmoteSheetSend.test.tsx` keeps the exception narrow.

**Touch.** Desktop hides the bare `+` until row hover; a phone has no
hover and a permanent `+` on every frame is the vertical spend the inline
row exists to avoid. So the gesture is **long-press the frame**
(`useLongPress`, branching on pointer TYPE rather than viewport width — a
touch laptop is real and a narrow window is not a phone). Existing chips
stay tappable where they sit; only the empty case needs the gesture. One
`EmoteSheet` for the whole transcript, mounted at `Terminal` — a sheet
per frame would stack N scrims and leave only the last reachable.

⚠⚠ **The client no longer holds its own copy of "what is reactable".**
`ReactionBar` carried a `REACTABLE_PREFIXES` array described as mirroring
`REACTABLE_TOPICS`. It had stopped: `act.combat` became reactable
server-side and the copy never learned, so combat frames silently offered
nothing — a drifted mirror looks exactly like a working one. The set now
rides `ConnectionEstablishedPayload.reactableTopics`, and an empty set
offers no affordance rather than guessing.

⚠ `REACTABLE_TOPICS` itself listed `'speech.vocal'` three times, a
leftover of the S2 collapse folding say/shout/whisper onto one name. A
`Set` deduped it, so nothing failed and the literal quietly claimed a
size it did not have. `reactableTopics.test.ts` asserts the **count**,
because a membership assertion passes either way.

**Not built, and it is a design question rather than an omission.** The
handoff's coalesced *"7 people reacted to what you said"* line for acts
you have scrolled away from. Where a client-composed sentence lives
without either impersonating server prose or reviving the notification
surface Wave 1C cut is unresolved. What ships instead: above the
threshold — where the server suppresses per-reaction prose — the counter
distinguishes your own acts (`✦ 12 reactions to yours`).

### Per-user controls

`SoulMixin.settings` defines five keys; each is **honored where its data
lives**, split between the server (delta-shaping) and the client
(pure render / transcript):

| Setting | Honored | How |
|---|---|---|
| `social.react.tagGroup` | **server** | `ReactionRegistry.buildActState` reads it per-recipient (`viewerPrefs` → `ShellApi.resolveSetting`); picks the bucket key — tag group vs per-verb. |
| `social.react.collapseThreshold` | **server** | the per-bucket `reactors` name list rides the delta only while a bucket's count ≤ this (else the chip shows just the count). |
| `social.react.intensity` | **client** | scales the chip/counter pulse animation (`off`→none … `vivid`→big). |
| `social.react.alwaysAggregate` | **client** | `App` drops reaction prose frames (those carrying `meta.inReactionTo`) from the transcript — the chip carries the aggregate. |
| `social.react.muteChannels` | **client** | `ReactionBar` renders nothing on `speech.channel` lines. |

The three client-honored prefs are resolved server-side at connect
(`ShellApi.resolveSetting`) and shipped in
`ConnectionEstablishedPayload.reactionPrefs` (the store seeds them in
`setConnected`, defaulting until the payload lands). There is no live
mid-session re-read — a settings change applies on the next connect.
The two server-honored prefs need no wire surface; they shape the
per-recipient delta directly. (The server still always emits both the
below-threshold line and the above-threshold delta — `alwaysAggregate`
is a client-side hide, not a server-side suppression.)

## What this build deliberately is NOT

- No parallel reaction dispatch — the reaction *is* an emote with a
  scope.
- No keying on `causingCommandId`.
- No reaction-of-a-reaction (the regress gate).
- No client-side summing (counts are authoritative).
- No renown score / aggregator / persistence (raw event only).
- No overlay rendering — this build makes reactions *consumable* by the
  overlay (the scope-keyed delta seam); scene selection / rendering are
  the livestream build's (see [livestream.md](./livestream.md)).
