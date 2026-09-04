# Livestream

The livestream subsystem is the server half of the **pbox-stream**
overlay integration — the read-only **broadcast feed** that pushes
public stream state to OBS browser-source overlays, plus the
`stream` operator verb that drives it. Phase 1 is deliberately minimal:
the smallest end-to-end proof of the overlay↔saxonberg loop (auth +
feed + `StreamState`), driving the standby-overlay countdown off the
server instead of a demo URL param.

> **Not to be confused with the `broadcast` verb.** The in-world
> `broadcast` verb ([comms.md](./comms.md) / one-to-many ESP) sends a
> message to every online player — a comms concept. The "broadcast"
> here is the **livestream broadcast feed**: an out-of-band, read-only
> projection to overlay clients. They share an English word, not a
> namespace (`BroadcastController` vs the backend `BroadcastFeed`
> service).

## The pieces

| Piece | Where | Role |
|---|---|---|
| `StreamState` | `obj/StreamState.ts` (singleton Idea) | Session-wide overlay state (`mode` + `awayUntil`). |
| `Events.StreamStateChanged` | `lib/events.ts` | Well-known event carrying the full snapshot on every mutation. |
| `BroadcastFeed` | `backend/BroadcastFeed.ts` (singleton) | Read-only projection: snapshot-on-connect, re-push-on-change. |
| `StreamStateEnvelope` | `@saxonberg/types` | Dedicated wire frame (`{ type: 'stream-state', state }`). |
| broadcast principal | `WebSocketService` + `Backend` + `Application` | `?broadcast=<token>` upgrade path; a pure push target. |
| `requiresStreamer` / `streamers` | `lib/command/validators` + `AccessRegistry` | Authorization axis for the control plane. |
| `stream` verb | `cmd/stream/` + `StreamController` | `stream away <dur>` / `stream back`. |

## The read-only broadcast principal

OBS browser sources can't set WebSocket headers, so the broadcast
connection authenticates with a `?broadcast=<token>` query param.
`WebSocketService.handleUpgrade` checks it **before** the normal
session-auth path (so a cookieless overlay isn't 401'd):

- `BROADCAST_TOKEN` (env) enables the feed iff set and **≥ 16 chars**;
  a set-but-weak token logs an error and leaves the feed disabled
  (fail-loud, not open). Enabled state is resolved once at
  `initialize()`; per-upgrade work is a constant-time compare
  (`crypto.timingSafeEqual`).
- On match, the upgrade is accepted as the sentinel principal
  `service:broadcast` and threaded through
  `Backend.handleWebSocketConnect(ws, userId, sessionId, isBroadcast=true)`
  → `Application.handleUserConnect(..., isBroadcast)`.

The broadcast connection has **no `Interactive` at all** — stronger
than "no Avatar / no CommandGiver". It's registered straight with the
`BroadcastFeed` as a pure push target and is absent from the connection
registry, so `processUserMessage` finds no holder and commands are
rejected by construction. It is never routed game traffic (DMs, private
channels). Disconnect flows through `Application.handleUserDisconnect`,
which calls `BroadcastFeed.removeConnection` (a cheap no-op for normal
sockets).

Security note: the token rides in the OBS source URL (private to the
streamer) and grants only public, read-only data — acceptable for the
query-param delivery. Rotate via the env secret.

## `StreamState`

A `PostRegistrationMixin(Idea)` singleton at `/platform/idea/StreamState`
(a platform-pack `boot:` entry; row `{ class: /platform/idea/StreamState, data: {} }`).
Two transient fields:

```ts
mode: 'live' | 'standby';   // 'standby' = countdown running
awayUntil: number | null;   // absolute epoch-ms; null when live
```

Mutation goes through `setAway(awayUntil)` / `goLive()`; both emit
`Events.StreamStateChanged` with the full `snapshot()`. `canDestruct`
refuses (system singleton). **In-memory only in Phase 1** — a reboot
resets to `live`; durable round-trip is deferred (no persistent fields,
mirroring `TopicCatalogue`).

Routing change through the event bus (rather than a direct
`StreamState → BroadcastFeed` call) keeps the mutator decoupled from
the feed and lets future mutators (Phase 3 scene / lower-third / afk
commands) reach the feed for free.

## `BroadcastFeed`

A `backend/` singleton, peer to `ConnectionManager`. Owns the set of
broadcast `socketId`s + a per-socket frame counter (broadcast
connections have no `Interactive`, so the feed owns the ordering
primitive `Interactive.nextFrameId` provides for normal connections).

- `addConnection(socketId)` — installs the `StreamStateChanged`
  listener lazily on first connect (well after boot, so the
  `EventRegistry` is live), then pushes the current snapshot.
- On `StreamStateChanged` → re-push the full snapshot to every
  connection (the state is tiny — no diffing).
- Reaches the wire via `Backend.sendEnvelopeToSocket` directly. The
  "game objects reach Backend only through Application" chokepoint
  governs mud/ objects; `BroadcastFeed` is a backend service at the
  same layer.

Beyond `StreamState`, the feed also forwards the **overlay owner's own
relay chat** (both platforms, unified) — the reaction-delta seam's sibling.
See [streaming.md](./streaming.md) for the full picture; the wire seam is:

- The relay ([`StreamRelay`](./streaming.md)) stays **mud-pure** — on every
  delivered line it emits `Events.RelayMessage` (`{service, channelKey,
  channelHandle, speaker, text}`) via `EventApi.emit`, never importing
  `backend/`. `BroadcastFeed.ensureSubscribed` installs an
  `EventApi.on(Events.RelayMessage)` listener (beside the reaction-delta
  one) that **filters to the owner's `OVERLAY_TWITCH_LOGIN` /
  `OVERLAY_YOUTUBE_CHANNEL`** (matched by the configured handle — a viewer
  tuning a *different* channel never matches, so nothing leaks) and pushes a
  **`RelayChatEnvelope`** (`{type:'relay-chat', frameId, service,
  channelHandle, speaker, text}` in `@saxonberg/types`) to every broadcast
  socket. The `pbox-stream` overlay renders it; this repo ships only the
  wire contract + the forwarding.
- The read is **overlay-presence-gated**: `addConnection`'s 0→1 edge calls
  `StreamApi.setOverlayReading(true)` (1→0 → `false`), which sentinel-tunes
  (`overlay:broadcast`) the owner's configured channels through the same
  readers used for viewer tuning — reusing the presence edge to open/close
  the owner reads independent of any player `tune`. YouTube re-resolves the
  owner's current live broadcast on a slow single-channel live-status poll
  (`youtube.overlayPollIntervalMs`, default 15 min — `search.list` is
  quota-costly) to catch a stream restart. One token / one feed / one owner
  is preserved — the owner's channels are env config, no per-overlay
  `StreamState` keying (the deferred multi-streamer non-goal).

## Authorization — the streamer axis

The control plane is gated on the orthogonal **streamer axis**, distinct
from the developer axis (a streamer drives the overlay without holding
TypeScript-escape capability). See [access.md](./access.md):
`AccessApi.isStreamer` + the bootstrap-seeded `streamers` managed group
+ the `requiresStreamer` validator.

Membership is seeded from the `STREAMER_PLAYER_IDS` env var
(comma-separated Avatar playerIds) at `AccessRegistry.postRegister` —
additive + idempotent (never removes), deploy-time config alongside
`BROADCAST_TOKEN`. Drop a member via the `group` verb.

## The `stream` verb

`cmd/stream/stream.yaml` + `StreamController` — one verb, two
subcommands (the `chat`/`alias` subcommand-dispatch shape):

- `stream away <duration>` — enter standby; the overlay counts down to
  `now + duration`. Duration is shell-style (`15m`, `1h30m`, `90s`; a
  bare number is minutes).
- `stream back` — clear standby, go live.

**Affordance vs authorization split:** the verb rides `AuthorMixin`'s
self command bucket (the operator command surface, like `eval`/`reload`),
but authorization is the `requiresStreamer` gate — an author who isn't a
streamer sees the verb but can't run it. The split lets a future
dedicated streamer-mode affordance narrow visibility without touching
the gate.

## What's NOT in this build (Phase 1)

- **Durable `StreamState`** across reboots (in-memory only).
- **Scene / lower-third / camera-focus / alert-queue** fields on
  `StreamState` (the rest of the control plane — Phase 3).
- **Public chat forwarding** over the feed (Phase 2 — taps only public
  channels).
- **The CLI / React streamer-mode / Stream Deck remotes** for the
  control plane (Phase 3).
- **`@saxonberg/mml` extraction** for prose-rich overlays (Phase 5).

The overlay (client) half lives in the sibling `pbox-stream` repo; the
full phased roadmap is in its `PLAN.md` and `docs/phase-1-server-spec.md`.

## Cross-references

- [access.md](./access.md) — the `isStreamer` predicate, `streamers`
  group, `requiresStreamer` validator.
- [connection.md](./connection.md) — the WS upgrade path the broadcast
  principal short-circuits.
- [comms.md](./comms.md) — the in-world `broadcast` ESP verb (the
  distinct concept this subsystem is *not*).
- [response-envelope.md](./response-envelope.md) — the `Envelope` wire
  union `StreamStateEnvelope` joins.

## History

- **Phase 1 (broadcast feed)** — `StreamState` + `StreamStateEnvelope`
  + `?broadcast=` auth + `BroadcastFeed` + `requiresStreamer`/`streamers`
  + the `stream` verb. saxonberg `feature/broadcast-feed` (commit
  `532725ce`, MR !72). The overlay side shipped in `pbox-stream`
  `feature/phase-1-broadcast`.
