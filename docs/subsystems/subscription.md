# The subscription protocol — one wire for every state push

**Built by:** navigation-server (2026-11). **Owner's rule:** *"if we need a
new push mechanism then do we still need the old one? … why not make one
mechanism that works for everything."*

A **subscription** keeps a copy of some state current on a client. Every
state push the server makes rides it: live MQL queries (the shelf, live
cards), forum boards, your client state, the press ticker, your map, and
where you are. **Events stay events** — a dispatch outcome, a prompt, an
engagement's progress, a reaction flush, a card being born or closed happen
once and have no copy to repair.

## The wire

```ts
// client → server
{ type: 'subscribe';   subscriptionId; source: string; params?: unknown }
{ type: 'unsubscribe'; subscriptionId }
// server → client
{ type: 'subscription-result'; subscriptionId; rev: 0;        snapshot }
{ type: 'subscription-delta';  subscriptionId; rev: number;   changes: SubscriptionChange[] }
{ type: 'subscription-error';  subscriptionId; reason: string; detail? }

interface SubscriptionChange { op: 'add' | 'replace' | 'update' | 'remove'; key: string; fields? }
```

- **`source` is a template path** (`SUBSCRIPTION_SOURCES` in
  `@saxonberg/types` names the engine's own). There is no list of sources
  anywhere: the registry loads one on use.
- **`rev`**: the snapshot is `0`, each delta `rev + 1`, per subscription. A
  socket neither drops nor reorders inside a connection and nothing here
  drops, so a break is a defect. **Repair is always the same act:
  subscribe again** — after a gap, a reconnect, a restart. A second session
  is simply its own subscriptions.
- Per-source shapes are typed in `SubscriptionSources` (params, snapshot,
  change fields), so the client and the harness narrow on what they asked.
- **Read-only.** The protocol never writes what a client asks it to
  (`backend/inbound/README.md`); `client-state-write` stays a request.

## The server

- **`SubscriptionRegistry`** (`platform/idea/`) holds, once, what each
  registry used to hold a copy of: the per-session id table, `rev`, the
  three emits, the disconnect sweep. Delivery is **per session**
  (`interactive.sendEnvelope`), never every session of a body.
- **The session's verbs are on `Interactive`**: `subscribe(id, source,
  params, { silent })`, `cancelSubscription`, `cancelAllSubscriptions`,
  `hasSubscription` (`lint:object-verbs`).
- ⭐⭐ **One Api: `SubscriptionApi`** (`api/subscription.ts`), over one
  logic singleton (`SubscriptionLogic`, `/platform/idea/api/subscription`).
  Its producer verb is `poke(source, holder | null, hint)` — a body, a
  ledger or a session telling a source its answer may have changed, the
  hint in the source's own words: the sections (where you are), a
  `{ durableSubject }` a ledger just wrote, a holder whose session changed
  bodies, a viewer's `{ belief }` about a subject (the MQL source). It also carries the subscribable-field
  substrate any source projects through (`SubscribableFieldDescriptor`,
  `fireFieldChange`, `projectFields`, `projectAspect`), the MQL source's
  one-shot `handleQuery`, and the sandbox reap (`cancelAllForScope`).
  ⚠ There was a second, `MqlSubscriptionApi`, with its own logic
  singleton and bespoke pokes (`notifyDurableSubject`,
  `refreshForInteractive`, and crowded-rooms' `notifyBelief`); it was
  folded in at review (MR !595) — a
  source is a source, and it does not get a facade of its own.
- **Finding a source.** A standing instance at the path is asked whether it
  is a source; otherwise the row's CLASS is read (`StuffApi.classForRef` —
  loads the module, instantiates nothing) and refused (`source`) unless it
  composes `SubscriptionSourceMixin`; only then is the singleton stood up.
  A client cannot make the server stand up an arbitrary singleton.
- **`SubscriptionSourceMixin`** (`lib/subscription/`): `openSubscription(sub,
  params)` answers the snapshot (synchronously when it can — a live card's
  open must have its answer in hand) or refuses in a word;
  `closeSubscription(sub)`; `pokeSubscriptions(holder, hint)`;
  `pauseSubscription(sub)` / `resumeSubscription(sub)`. A source
  holds a `Subscription` handle — `push(changes)`, `fail(reason)`, `end()`,
  `state` — and never builds an envelope or keeps an id table.
- ⭐ **Pause and resume** (crowded-rooms D4/D5, ported onto the protocol at
  the merge). `subscription-pause {subscriptionId}` says a subscription is
  NOT ON SCREEN — a card scrolled away, minimized, behind a `full` card, the
  tab hidden; the client says it, the server never infers it. The registry
  drops everything pushed while paused and tells the source
  (`pauseSubscription(sub)`), which may also stop computing (the MQL source
  drops its dependencies and memo). `subscription-resume` asks the source
  for a fresh snapshot (`resumeSubscription(sub)`; the default closes and
  reopens with the params it was opened with) and sends ONE
  `subscription-result` at `rev` 0 — a reset the client replaces its copy
  with — then deltas from `rev` 1. Session verbs: `pauseSubscription`,
  `resumeSubscription`, `isSubscriptionPaused` (a card's rebind keeps a
  pause). Unknown ids are skipped.
- **Silent opens.** A live card (`CardRegistry`) and the two session
  subscriptions open silently: the snapshot rides their own envelope
  (`card-opened`; `connection-established.clientState` / `.releaseWindow`),
  and changes ride `subscription-delta`.

## The sources

| source | params | snapshot | change key | who pokes |
|---|---|---|---|---|
| `MqlSubscriptionRegistry` | the query (`chrome: 'self'` for the shelf) | records | the record | wakes itself (its dependency index) |
| `ForumSubscriptionRegistry` | `{ scope }` | `{ scope, records }` | the entry | wakes itself (the forum ledger) |
| `ClientStateSource` | — | the holder's client state | the key | `pushClientStateUpdate` |
| `PressFeedSource` | — | the live window | the releaseId | the press's mutators (`null` holder) |
| `MapCopySource` | `{ locality? }` | your map, claims tagged | `<locality>\|<claim key>` | `NavigationApi.recordPlace`, by owner key (one save per document per beat) |
| `WhereaboutsSource` | — | `{ sections }` | the section's name | wakes itself — its declared table, `WHEREABOUTS_WAKES` — plus the clock and a moving place |

A new kind of state is a new SOURCE — a class composing the mixin, and a
row — never a new message type and never a kernel list edit.

## Where you are and your map (navigation-server)

- **`WhereaboutsSource`** keeps one flush per beat per holder: a wake marks
  sections dirty and arms one `ScheduleApi.schedule(0, …)`; the flush calls
  `Perceiver.navigation(names)` once and sends each subscription only the
  sections that differ from what IT was last sent, each section stamped `v`
  + 1. A failed read is absent and sends nothing. The horizon is sent on
  arrival and in a snapshot only.
- ⭐⭐ **What wakes it is ONE declared table, `WHEREABOUTS_WAKES`**
  (`platform/idea/WhereaboutsSource.ts`; navigation-server review). A
  section re-reads when something it showed fires a field change — the
  same `FieldChangedEvent` a live card wakes on — and each row says whose
  field (your **body**, your **place**, a **way** out or its door,
  something you **carry**), which field, which sections, and why in a
  player's words:

  | whose | field | sections | why |
  |---|---|---|---|
  | body | `container` | all | you moved |
  | body | `contents` | you · ways · place | what you hold or carry changed — hands, load, a key, a lamp, a lead |
  | body | `posture` | you · ways · place | you sat, stood, lay or stooped |
  | body | `engagements` | you · ways | a fight or an act began or ended |
  | body | `hauling` | you · ways | you hitched or unhitched a cart |
  | body | `occupants` | you · ways | someone mounted you, or got down |
  | body | `settings` | you | your pace |
  | place | `contents` | place | who and what is here — the crowd, the light |
  | place | `name` | place | the place was renamed |
  | way | any | ways | a door opened, shut or locked; a gangway bolted |
  | carried | `on` | you · place | a lamp you carry was lit or put out |

  Two wakes no field carries, and no command causes, poke it instead: **the
  clock** (the weather's segment boundary → every holder's `place`) and **a
  place that moves under you** (a ship's voyage, a river's reach →
  `localities`, `zone`, `place`). ⛔ **Nothing else.** It used to be poked
  by hand from six places, among them after EVERY command you ran — so
  `say` recomputed the ways (every exit's full gate check) and the panes
  changed for reasons nobody could name. Adding something a pane shows is
  a row here and a `fireFieldChange` at its setter (doors, posture,
  engagements, settings and hitching fire for this table — and so a live
  card showing them wakes too). `Bistate.setState` fires `open`/`on`/
  `folded`, so a door opened by the VERB is heard (the setter override the
  table replaced was never called by `open door`).
- ⚠ **The projection helpers are the descriptor's, not a method's.**
  `SubscriptionApi.perceivedAmong` / `noteRead` / `sharedRead` admit only
  the live projection's own frame (a descriptor's `read` lambda) and the
  MQL source. A `Stuff` method is a proxied call that pushes its own frame,
  so a descriptor that calls `place.someMethod()` which then calls them is
  REFUSED — the projection fails and the card silently stops waking (cost
  a bisection on `present`). Keep the rule in a module-private function
  both the method and the descriptor call, and pass the live filter in
  from the descriptor (`Place.ts`'s `peopleHere`).
- ⭐⭐ **Every change names its cause.** A `subscription-delta` carries
  `causes`: the ids of the command(s) whose work produced it — the same id
  as the prose that command sent (`meta.commandId` /
  `meta.causingCommandId`). A source that pushes later than it learns
  (the MQL source, where-you-are) calls `sub.noteCause()` where it marks
  dirty, because only there is the command in the execution context; a
  push reads it again itself. The client ties the change to the line
  (`ChangeMark`, card-surface.md): *changed by: <the line>*, *you didn't
  see what changed this* (a command, but no line of it reached you — the
  perception gate comes free), or *time passed* (no command: the clock).
- **`MapCopySource`** is always the map of whoever the session drives (its
  `mapOwnerKey()` — a shade reads its person's); nobody else's can be asked
  for. Every path on the wire is a `TaggedPath` (`{ path, spoiler: 1 }`);
  a Locality's address travels bare.
- The client opens both at session start and replays them (unsubscribe,
  then subscribe) on every connection; the `map` card is a client body over
  the sections (`client/src/store/navSlice.ts`).

## What you can type here (the which build)

`WhichSource` (`/platform/idea/WhichSource`, `SUBSCRIPTION_SOURCES.which`)
keeps a `which` card current. Params `{ what? }` — the word the card asked.
It wakes on the command stack (the three sealed `CommandGiver` mutators poke it)
and on a `FieldChangedEvent` on the body, its place, or any source on its stack;
one flush per beat per holder; it re-reads `CommandApi.which` and pushes
`{ op: 'replace', key: 'answer', fields: <the whole answer> }` only when the
answer's JSON changed. A word that names nothing any more answers empty with its
ask kept. Shape: `WhereaboutsSource`'s.

## History

- **navigation-server (2026-11)** — the protocol replaced
  `mql-subscribe`/`-result`/`-delta`/`-error` and their five forum twins,
  `client-state-update` (and its boot-time DI seam), the press ticker's
  `publication.press` frames and the navigator card's payload push
  (`refreshNavigator`).
