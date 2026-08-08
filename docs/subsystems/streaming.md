# Streaming — the unified livestream tuning surface

One platform-agnostic surface for **tuning into a livestream** — any live
broadcast on **Twitch, YouTube, or Kick** — and consuming it inside the
cockpit: **watch** its video in the focal embed, **follow** its chat in the
aggregated chat pane, and (Twitch this cycle) **post** to that chat under
your own linked identity. The streamer is named by an argument string (a
URL, a handle + a platform opt, or an in-game character), resolved lazily
into a runtime `StreamerTarget` — **nothing is persisted**.

This subsystem unifies three formerly-separate surfaces: the shipped
**Twitch chat relay** (folded in here — see the superseded banner atop
[twitch-relay.md](./twitch-relay.md)), the **livestream-viewer video embed**
(was an operator-curated `broadcastSources` picker, now `watch`-driven —
[cockpit-layouts.md](./cockpit-layouts.md)), and **YouTube chat** (new,
read-only). The **player-facing surface is one code path**; only the
backend *transports* stay per-platform because the protocols genuinely
differ. **Kick** (read-only) joined as the third transport — the first
**inbound-webhook** transport (Kick pushes signed HTTP POSTs; there is no
outbound socket or poll) — alongside Kick becoming a co-equal **auth
provider** (`KickProfile`, login + link; see
[connection.md](./connection.md)).

It also extends the [broadcast overlay](./livestream.md): the single overlay
owner's OWN active-stream chat (both platforms) is forwarded onto the
`BroadcastFeed` while live.

## The two verbs — `watch` (single) vs `tune` (multi)

Two top-level verbs in the `mud/cmd/stream/` category (beside the
overlay-owner `stream` verb), because the cardinalities are a genuine
semantic difference:

- **`watch <target>`** — the focal **video** embed (cardinality **1**). Sets
  the server-authoritative per-viewer `cockpit.watch` clientState; a second
  `watch` swaps it; `watch off` clears it. `watch` also **implies** `tune`
  (watching a stream pulls its chat in).
- **`tune <target>`** — **chat** follow (cardinality **N**). Follows a
  streamer's chat into the aggregated pane; many at once, interleaved with
  per-line provenance. `tune <target> <message>` posts (Twitch, linked
  account). `tune off <target>` stops; bare `tune` lists; `tune
  list`/`who`/`history` behave platform-agnostically.

Both are `WatchController` / `TuneController` (`obj/command/stream/`) over
one `StreamApi`.

## The target grammar — opt, not prefix

`StreamerTarget.parse` (`lib/streaming/StreamerTarget.ts`) is **pure, total,
and unit-tested** — no network / DB. It classifies `arg` + the mutually-
exclusive `--twitch` / `--youtube` / `--kick` boolean opts into a
`ParsedTarget`:

1. **URL** — `twitch.tv/shroud`, `youtu.be/…`, `youtube.com/@handle` /
   `watch?v=…` / `live/…` / `channel/UC…`, `kick.com/<slug>` (slug-only —
   video/clip URL forms are out of scope). Platform + identifier parsed
   from the URL; a **conflicting** platform opt → `url-opt-conflict`
   reject.
2. **Bare handle + opt** — `shroud --twitch`, `mkbhd --youtube`,
   `xqc --kick` (slugs lowercased). A bare handle with **no** opt and
   **multiple** opts are rejects (`ambiguous-handle`). No
   `platform:handle` prefixed strings — the platform rides in the URL or
   the opt, never smuggled into the arg.
3. **Character / MQL** — `tune alice`, `tune @alice` (no opt). Resolved
   **Twitch-first, Kick-fallback**: `Avatar → User → TwitchProfile.login`,
   else `User → KickProfile.slug` (a character linked to both resolves
   Twitch — the two-way platform; deterministic). `unlinked` only when
   neither link exists. A `@`-seed with `--youtube` is a
   `character-youtube` reject (deferred non-goal — `GoogleProfile` stores
   no channel).

`StreamerTarget.classifyYoutubeRef` (also pure) sub-classifies a YouTube
identifier into `videoId` / `channelId` / `handle` — the shared branch point
for both the `watch` embed shape and the chat resolve.

The **resolved** `StreamerTarget` value (`{platform, key, handle,
character?}`) is produced by `StreamLogic` (the async resolver — the network
/ DB half); `key` is the transport's stable channel key (Twitch/Kick
broadcasterId / YouTube liveChatId).

## Module layout (CLAUDE.md taxonomy)

| Piece | Where | Role |
|---|---|---|
| `StreamerTarget` | `lib/streaming/StreamerTarget.ts` | Named value-object + the pure parser/classifier. Backend-free. |
| `StreamRelay` | `obj/StreamRelay.ts` (`/obj/StreamRelay`) | In-memory `Idea` singleton — the channel table + caches. Pure mudlib (EventApi.emit only, no backend import). |
| `StreamLogic` | `obj/api/StreamLogic.ts` (`/obj/api/stream`) | Gated logic singleton (`FromModule('/api/stream#StreamApi')`). The async resolver + the Api→backend bridge. |
| `StreamApi` | `api/stream.ts` | Thin forwarding facade + the call-shape types. `decorateApiClass`. |
| `TwitchClient` / `TwitchRelayReader` | `backend/` | Twitch transport (EventSub multiplex + Helix). |
| `YoutubeClient` / `YoutubeRelayReader` | `backend/` | YouTube transport (per-`liveChatId` read). |
| `KickClient` / `KickRelayReader` | `backend/` | Kick transport (app-token subscriptions; inbound via the webhook). |
| `KickWebhookVerifier` / `KickWebhookRoutes` | `backend/` | The signed-inbound receiver (see below). |
| `tune` / `watch` | `mud/cmd/stream/*.yaml` + `obj/command/stream/*Controller.ts` | The verbs. |

There is **no** `mud/api/youtube.ts` / `YoutubeLogic` (nor a kick
sibling) — the surface is unified on `StreamApi`; only the *transport* is
per-platform.

## The relay state — `StreamRelay`

Player-initiated, memory-resident, lazy (no registry collection; evaporates
on reboot). The channel table is keyed by a composite
`channelKey(service, key)` so Twitch + YouTube + Kick channels coexist in one table
read by the platform-agnostic `list`/`who`/`history`. Each entry carries its
`service` + display `handle`. Mutators (`addTuned` / `removeTuned` /
`dropPlayer` / `dropChannel`) return the presence **edges** (0→1 / 1→0); the
`StreamLogic` bridge reads the edges and drives the right backend reader's
subscribe/unsubscribe. Delivery reuses the lone `MessageApi.sendMessage`
chokepoint (external speakers have no `Stuff` actor, so not `.scene`) on the
`world.<service>.message` topic — subscription-gated, not implant-gated.

`deliver` also emits `Events.RelayMessage` per line (the overlay-forwarding
seam — see below). The **overlay sentinel** (`overlay:broadcast`) is
`addTuned`-able to a channel so a reader opens with no real player tuned; it
counts toward the presence edge but is guarded out of `whoTuned` /
`tunedChannelsFor`.

## The transports (per-platform, behind the unified Api)

**Twitch** — `TwitchClient` holds **one** EventSub websocket session and
multiplexes N `channel.chat.message` subscriptions over it; `TwitchRelayReader`
is the presence-gated subscribe/unsubscribe worker + Helix login→id resolve +
Send Chat Message (outbound). One operator reader account
(`TWITCH_READER_USER_ID`, `user:read:chat`) reads any public channel. The
three-case `RelaySpeaker` identity bridge (in-game egress mirror /
external-unlinked / external-linked persona-on-hover), echo-suppress, and the
token-bucket throttle all live here (the Twitch relay's grandfathered dials).

**YouTube (read-only)** — no multiplexed session; each `liveChatId` is its
own long-lived read. `YoutubeClient` (`backend/`) owns:

- **Reader auth** — a single operator reader account via env
  (`YOUTUBE_READER_CLIENT_ID` / `_SECRET` / `_REFRESH_TOKEN`), refreshed
  against Google's token endpoint. **No `GoogleProfile`, no per-player
  tokens** (posting is deferred), and **no separate `YOUTUBE_API_KEY`** — the
  reader token covers both chat reads and the `channels.list` resolve.
- **Resolution** — `@handle` → channelId (`channels.list?forHandle`, 1 quota
  unit, **memoized** — the single home consumed by both `tune` and `watch`);
  channel/video → the current live broadcast's `activeLiveChatId`
  (`search.list?eventType=live` → `videos.list?part=liveStreamingDetails`).
- **Streaming** — the `list`-poll path (wired day one behind the transport
  seam; `streamList` is a later optimization) at the server's
  `pollingIntervalMillis` (floor `youtube.pollIntervalMs`), delivering each
  line + the stream-end signal through the injected transport (tests drive
  synthetic chat).

`YoutubeRelayReader` is the presence-gated per-`liveChatId` reader:
`isConfigured`, `resolveChannelId` (the `watch` embed path), `resolveChannel`
(→ live `activeLiveChatId`, **live-only** — `not-live` reject),
`subscribe`/`unsubscribe`, and **stream-end auto-untune**: when the read
ends, `StreamApi.dropChannel('youtube', liveChatId)` drops the relay entry
and notices the tuned players ("the stream ended").

### YouTube boundaries this cycle

- **Read-only.** `tune … --youtube <message>` → `read-only` reject. Outbound
  posting (`youtube.force-ssl`, per-player Google OAuth, the quota
  accountant) is deferred; the CLI reserves the shape.
- **Character → YouTube** is deferred (no channel on `GoogleProfile`).
- **Live-only bind**; a *new* broadcast on an already-tuned channel means
  re-tuning (viewer auto-rebind deferred; the overlay owner gets a slow
  live-status poll instead).
- **Dials are `youtube.*` AppSettings** (historyCap, reconnectBackoffMs,
  pollIntervalMs, overlayPollIntervalMs) — not hardcoded constants.

### Kick (read-only, the webhook transport)

The third transport inverts the shape: Kick's official API delivers chat
as **event subscriptions pushed to a registered public HTTPS webhook** —
the first inbound receiver in the streaming stack. Pieces:

- **`KickClient`** (`backend/`) — the outbound half: the **app access
  token** (client-credentials grant, cached; Kick allows app-token
  subscriptions with an explicit `broadcaster_user_id`, so the read path
  needs **no user token at all**), slug → broadcaster-id resolve
  (`GET /channels?slug=`, TTL-cached via `kick.resolveCacheTtlMs`),
  `chat.message.sent` subscription create/delete, and the webhook
  public-key fetch.
- **`KickWebhookRoutes`** (`POST /webhooks/kick`) — mounts in
  `Server.setupMiddleware` **before the global body parser and before
  session/passport**: signature verification needs the byte-exact raw
  body (`express.json()` consumes the stream irrecoverably), and the
  early mount makes the route structurally session-free. Route-local
  `express.raw`; 404 when the transport is unconfigured (dormant, no
  info leak).
- **`KickWebhookVerifier`** — verification precedes ALL parsing:
  header presence → **RSA-PKCS1v15/SHA256** over the period-joined
  `messageId.timestamp.rawBody` against Kick's published public key
  (`GET /public-key`, cached, ONE re-fetch-and-retry on failure — the
  key-rotation hedge) → replay window (`kick.replayWindowSec`, both
  skews) → **ULID dedup ring** (`kick.dedupTtlSec` / `kick.dedupMaxSize`
  — Kick delivers at-least-once). `duplicate` is acked 200 (acking stops
  retries) but never re-dispatched; everything else non-ok → 401 + a
  warn carrying only the reason + messageId (never the body or
  signature).
- **`KickRelayReader`** — the presence-gated subscription worker:
  create on the relay's 0→1 tuned-player edge, delete on 1→0, over a
  memory-only `broadcasterId → subscriptionId` map (a crashed process
  can orphan a subscription on Kick's side — harmless, deliveries for
  un-tuned channels drop at the relay's channel-key miss; a boot-time
  reconciliation sweep is a named deferred seam). Verified inbound is
  normalized → `StreamApi.dispatchInbound('kick', …)`.

**Binding is persistent, Twitch-style** — no live-only bind, no
stream-end auto-untune (Kick chatrooms outlive broadcasts; the embed
renders offline channels). Kick boundaries this cycle: **read-only**
(`tune … --kick <message>` → `read-only` reject; posting = the phase-2
`kick-reauth` + `chat:write` seam), and local dev runs
**transport-dormant** (`isConfigured` gates on `KICK_CLIENT_ID` /
`KICK_CLIENT_SECRET` / `KICK_WEBHOOK_URL`; the verifier + route are
keypair-unit-tested, so no dev tunnel is needed).

## The `watch` embed

`watch` writes `cockpit.watch` (`WatchTarget | null`, transient clientState)
and pushes it; the client mirrors it and renders **one** sandboxed iframe (no
picker — the verb is the control). Most forms resolve with **no external
call** (Twitch handle, Kick slug — `player.kick.com/<slug>`, muted
autoplay, renders offline channels gracefully — YouTube URL / videoId /
`UC…` channelId), so watching works even when the chat relay is
unconfigured. A bare YouTube **`@handle`**
resolves to a durable `{platform:'youtube', channelId}` via
`StreamApi.resolveYoutubeChannelId` (reusing the reader credential, memoized;
reader-unconfigured → reject-and-point at a URL) — rendered
`live_stream?channel=<channelId>` so it tracks the channel's live status.
Watching also best-effort delegates to `StreamApi.tune` (Twitch now; the
YouTube implied chat-tune resolves live-or-not through the same path). See
[cockpit-layouts.md](./cockpit-layouts.md) for the client half.

## Identity / rendering

`RelaySpeaker.service` spans `'twitch' | 'youtube' | 'kick'`. Twitch and
Kick lines from a linked speaker render `external-linked` (handle default,
persona-on-hover via the reverse `TwitchProfile`/`KickProfile → User`
walk); YouTube lines are `external`-only (no channel stored → no reverse
link this cycle). The client `relayTemplate` (parameterized by service —
Twitch-purple / YouTube-red / Kick-green glyph) renders
`speech.relay` (one topic; the platform rides `payload.service`);
message text is escaped plain text, never MML (untrusted external
input).

## Overlay chat forwarding (the owner's own chat)

The single overlay owner's OWN external channels are env config
(`OVERLAY_TWITCH_LOGIN` / `OVERLAY_YOUTUBE_CHANNEL` /
`OVERLAY_KICK_CHANNEL`, beside `BROADCAST_TOKEN` /
`STREAMER_PLAYER_IDS`). While ≥1 `service:broadcast` overlay is connected,
`BroadcastFeed` drives `StreamApi.setOverlayReading(true/false)` on the
0→1 / 1→0 presence edge, which **sentinel-tunes** the owner's channels
through the readers (reusing the presence edge). The relay emits
`Events.RelayMessage` per delivered line; `BroadcastFeed` filters to the
`OVERLAY_*` channels (by the configured handle — a viewer tuning a different
channel never matches) and pushes a **`RelayChatEnvelope`** to the broadcast
socket. The relay stays mud-pure (event bus only, no backend import); the
overlay *render* lives in the sibling `pbox-stream` repo. One token / one
feed / one owner is preserved. Full seam in [livestream.md](./livestream.md).

## Presence-gating, invariants

- A channel's backend read exists only while ≥1 player (or the overlay
  sentinel) is tuned (0→1 opens, 1→0 closes; Twitch debounced; YouTube per
  `liveChatId`; Kick per webhook subscription, undebounced). `PlayerLoggedOut`
  drops a player from every channel; the drop + per-transport unsubscribe is
  **centralized in `StreamLogic.dropPlayer`** (so one observer — the Twitch
  reader — covers all three readers).
- **Actor from context** — the posting player is `context.commandGiver` /
  `getActingAuthor`, never a parameter.
- **No hardcoded streamer list** anywhere except the single overlay-owner
  account.

## Deferred / non-goals

YouTube + Kick outbound (posting — Kick's is the `kick-reauth` +
`chat:write` phase-2 seam through the existing throttle/echo-suppress);
character→YouTube; Kick boot-time subscription reconciliation;
`kick.com/video/…` URL forms; the outbound platform-action
family (`follow`/`sub`/`cheer`/`gift`) + monetization; go-live notifications
& follow-groups; persisted tunes / discoverability; multi-streamer overlay
scoping; YouTube stream-start auto-rebind while tuned. See the requirements
for the full non-goal list.

## Cross-references

- [twitch-relay.md](./twitch-relay.md) — the Twitch-relay detail this folds
  in (superseded banner points here)
- [livestream.md](./livestream.md) — the overlay `BroadcastFeed` +
  `RelayChatEnvelope` seam
- [cockpit-layouts.md](./cockpit-layouts.md) — the `watch`-driven
  `cockpit.watch` embed
- [connection.md](./connection.md) — the OAuth spine + `TwitchProfile` /
  `GoogleProfile` / `KickProfile`
- [messaging.md](./messaging.md) — the `sendMessage` chokepoint
- [belief.md](./belief.md) — the persona the external-linked speaker reveals

## History

Built in six phases on `feature/stream-tuning` (P0 shared seams → P1 unify
chat + retire `twitch` → P2 `watch` + retire `broadcastSources` → P3
read-only YouTube → P4 overlay chat forwarding → P5 docs). Requirements +
plan: `docs/requirements/stream-tuning-requirements.md`,
`docs/plans/stream-tuning-plan.md`. The **Kick transport + the Kick auth
provider** landed on `feature/kick-relay` (vocabulary → provider →
client/verifier → relay wiring → docs; the first webhook-inbound
transport).
