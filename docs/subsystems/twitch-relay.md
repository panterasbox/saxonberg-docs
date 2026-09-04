# Twitch relay

> **Superseded — folded into [streaming.md](./streaming.md).** The Twitch
> relay is now one transport behind the unified `tune`/`watch` surface: the
> `twitch` verb is **retired** (platform rides in the target/opt), the
> `TwitchRelay`/`TwitchLogic`/`TwitchApi` classes are renamed
> `StreamRelay`/`StreamLogic`/`StreamApi`, and the channel table is keyed by
> a composite `channelKey(service, key)`. The `speech.relay` topic
> and the `TwitchClient`/`TwitchRelayReader` **transport** are unchanged.
> Read [streaming.md](./streaming.md) first; this doc is retained for the
> Twitch-specific transport detail (EventSub session, Helix, the
> incremental-scope reauth flow, the three-case identity bridge).

The two-way bridge between in-game players and a streamer's **Twitch
chat**. A player tunes into a Twitch channel by handle, reads its chat in
the cockpit, and posts to it under their own linked Twitch identity. It is
a **dedicated, separate surface** — its own `twitch` verb and its own
`speech.relay` topic — **not** a chat [`Channel`](./chat.md):
Twitch chat is an OOC/meta spectator layer, not an in-world frequency, and
folding it into `Channel` would scatter a strategic integration across an
unrelated subsystem and force the in-world chat path to carry external
special-casing forever.

This is the interactive-broadcast half of the
broadcast-patronage track
(Phases 2 + 6) and the live consumer of the
[auth-providers](../subsystems/connection.md) Twitch spine. It ships the
chat bridge; the live-stream **video** overlay is a later client piece.

## Player-initiated, memory-resident, lazy

There is **no registry collection and no persistent tuning.** Channels are
discovered at runtime and held only in memory:

- A player runs `twitch tune <handle>`. The login is resolved to a Twitch
  broadcaster id **once** (Helix `/users`), cached, and a lightweight
  channel entry is created on demand.
- The relay holds `Map<broadcasterId, { login, tuned: Set<playerId> }>` in
  memory on the `TwitchRelay` singleton. It evaporates on reboot; the next
  login + tune re-initializes everything lazily.
- A player is dropped from every channel on **logout**, so a channel's
  EventSub subscription is torn down once nobody in-game is listening.

The only durable Twitch state in the system is the per-user `TwitchProfile`
(OAuth identity + tokens) the auth spine already owns.

## The layers and the dependency direction

| Piece | Home | Role |
|---|---|---|
| `TwitchClient` | `backend/` | The Twitch *transport*: the EventSub websocket session, Helix REST (subscription create/delete, Send Chat Message, `/users` lookup), and token refresh. Network + external I/O — infrastructure, modeled on `BroadcastFeed`. Event-type-parameterized, so the deferred payment-intake build reuses the same session/token plumbing. |
| `TwitchRelayReader` | `backend/` | The worker that owns the active EventSub subscription set (`broadcasterId→subId`) + per-channel debounce, wires inbound notifications to the mudlib, and exposes `subscribe`/`unsubscribe`/`resolveLogin`/`send`/`isConfigured`. Observes `PlayerLoggedOut` to unsubscribe emptied channels. |
| `TwitchRelay` | `mud/platform/` | The in-memory state singleton (`Idea`): the channel table, history ring, echo-tag store, send throttle, and the `deliver` fanout. **Pure mudlib** — no backend import, no events. Its tune/untune/`dropPlayer` mutators *return* the presence edges (`0→1` / `emptied`). |
| `TwitchLogic` / `TwitchApi` | `mud/platform/idea/api/`, `mud/api/` | The gated logic + facade. `TwitchLogic` is the **Api→backend bridge**: it imports `TwitchRelayReader` and calls it directly for resolve / send / subscribe / unsubscribe. |
| `TwitchController` | `mud/platform/idea/cmd/social/` | The `twitch` verb (`list` / `tune` / `untune` / `history` / `who` + bare post), with the outbound reject-and-point. |

**The dependency directions:**

- **Inbound** (Twitch → game) is **backend → mud**: the reader normalizes a
  `channel.chat.message` and calls `TwitchApi.dispatchInbound(...)` — a
  plain legal down-call (`backend/` may import `mud/`).
- **Outbound / resolve / subscribe** are **mud → backend**: initiated by a
  verb but needing Twitch. These go through the **Api bridge** —
  `TwitchLogic` imports the backend reader and calls it directly. This is
  the sanctioned pattern (the Api layer may import `backend/`, e.g.
  `ConnectionLogic`→`ConnectionManager`, `PersistApi`→`PersistenceManager`);
  the deep domain (`lib/`, non-api `obj/`) stays backend-free, which is why
  `TwitchRelay` itself never touches backend.

> **No port, no presence event.** An earlier shape inverted the mud→backend
> dependency through an injected `TwitchRelayPort` and a
> `TwitchPresenceChanged` event. Both were unnecessary once the Api-imports-
> backend rule was applied: they were two copies of the same workaround for
> a call that's actually permitted. They were removed in favor of direct
> bridge calls; the only event-bus use left is `PlayerLoggedOut`, which is a
> genuine engine-wide broadcast. See *History*.

## Incremental Twitch chat scopes

The auth spine deliberately grants **identity-only** scope
(`user:read:email`) at login/link. The relay needs two more, acquired on
demand through a re-consent flow this build adds:

- A **`twitch-reauth` Passport strategy** (own callback, `force_verify`) +
  `GET /auth/twitch/reauth?scope=<allowed>` re-runs OAuth for the current
  user with an **incremental** scope set (identity + already-granted +
  requested) and writes the broadened token back to the **existing**
  `TwitchProfile` via the same `handleProviderLink` upsert (a same-user
  re-consent returns `already-linked`, which is success). `TwitchProfile.
  hasScope` reads the result.
- **Reader account** — a single operator Twitch identity (env
  `TWITCH_READER_USER_ID`) holds `user:read:chat`; its id is the `user_id`
  condition on every EventSub subscription (Twitch lets one user read any
  public channel's chat) and powers the `/users` handle lookup.
- **Posting player** — each poster holds `user:write:chat`; an unlinked or
  unscoped poster is **rejected-and-pointed** at the reauth flow.

## The frame: identity bridge + provenance

Relay frames ride the relay's own topic `speech.relay` (a distinct
genre, so the cockpit cards them separately) and are **subscription-gated,
not implant-gated** — they carry no `modality`, so they reach every tuned-in
Avatar regardless of in-world implant. Delivery hand-builds the frame and
goes through the lone `onMessage` chokepoint (never
`MessageApi.scene`, which requires a Stuff actor — a Twitch viewer has
none).

The `payload.speaker` is a **three-case discriminated union** (in
`@saxonberg/types`):

1. `in-game` — a `StuffRef`: the outbound mirror of a local player's post.
2. `external` — a Twitch handle only: an unlinked viewer.
3. `external-linked` — both the Twitch handle **and** a `StuffRef` to the
   linked Avatar (when that player is online).

Rendering is **honest-to-origin**: a Twitch line shows the handle + a
`<twitch/>`-style provenance chip by default, and the linked MUD persona is
*revealed* on hover (case 3) — never substituted. An outbound post is sent
to Twitch first and **mirrored in-game only on success**, with an egress
marker; the reader's echo of that same `(twitchUserId, text)` is dropped
via a short-TTL **tag-and-suppress**. Outbound also carries a per-player +
global **token-bucket throttle** (bursts are shaped, not errored).

## Client

`twitchTemplate` routes `speech.relay` to a distinct treatment (a
Twitch-purple provenance chip + handle-default + persona-on-hover + egress
marker); the chip previews `twitch tune <login>` in the command bar on
hover and submits on click (the global clickable-previews-its-command
rule). Twitch message text is rendered as plain (escaped) text — untrusted
input never passes through the MML renderer.

## Operator setup

Set `TWITCH_READER_USER_ID` to an operator Twitch account and grant it
`user:read:chat` via `/auth/twitch/reauth?scope=user:read:chat`. That one
account powers both the handle lookup and the chat read for every relayed
channel. Until it is configured, `twitch tune` returns "relay not
configured" and nothing subscribes.

## Deferred

- **YouTube** and any second service — `TwitchClient` is Twitch-specific;
  the external-chat-relay slate
  holds the generalization (live-only + quota constraints).
- **Live-stream video overlay** — the client video half of "tune into a
  streamer"; this build is the chat half.
- **Recognition-driven name substitution** — v1 is origin-honest +
  hover-reveal; swapping the shown name by the viewer's belief state is a
  noted seam.
- **AppSettings dials** — history cap / throttle / echo-TTL are constants
  (`twitch.*` AppSettings candidates).

## Related

- [connection.md](./connection.md) — the Twitch OAuth spine + `TwitchProfile`
- [chat.md](./chat.md) — the in-world `Channel` surface this deliberately is *not*
- [messaging.md](./messaging.md) — the `onMessage` chokepoint the relay reuses
- [belief.md](./belief.md) — the viewer-aware naming the recognition seam would consume
- [livestream.md](./livestream.md) — `BroadcastFeed`, the backend-singleton-worker precedent

## History

- **Built 2026-06-28** (`feature/twitch-relay-build`). Shipped across:
  Phase 1 incremental-scope reauth, Phase 2 `TwitchClient`, Phase 3 relay
  surface, Phase 4 client. Then two design corrections landed before merge:
  (a) **player-initiated memory-resident channels** — dropped the
  `twitch_channels` Mongo collection + `TwitchChannel` Document + seeder +
  `TwitchTunedMixin` in favor of in-memory lazy resolution (`302cd31f`);
  (b) **direct Api→backend bridge** — dropped the `TwitchRelayPort`
  injection and the `TwitchPresenceChanged` event for direct
  `TwitchLogic`→`TwitchRelayReader` calls, once the Api-imports-backend rule
  was recognized (`ec08eecc`).
