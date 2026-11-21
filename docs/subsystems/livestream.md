# Livestream

The livestream subsystem is the server half of the **Rodney** overlay
(`packages/rodney`) and of a streamer's own controls: **the stream a
played body is running** (`BroadcastMixin`), the `stream` verb that runs
it, the per-streamer **broadcast feed** that pushes an allowlisted show to
the streamer's OBS browser sources, the **control door** pblive's Stream
Deck uses, and the two things that protect the people a broadcast shows —
**redaction** and the **on-air marker**. Built by broadcast-ready
(2026-11); Phase 1 (MR !72) was the single in-memory `StreamState`, which
retired.

> **Not to be confused with the `broadcast` verb.** The in-world
> `broadcast` verb ([comms.md](./comms.md) / one-to-many ESP) sends a
> message to every online player — a comms concept. The "broadcast"
> here is the **livestream**: an out-of-band, read-only projection to
> overlay clients. They share an English word, not a namespace — which is
> also why the mixin is `BroadcastMixin` (the stream a body carries) and
> the streamer AXIS is `AccessApi.isStreamer` (the `streamers` group).

## The pieces

| Piece | Where | Role |
|---|---|---|
| `BroadcastMixin` | `lib/livestream/Broadcast.ts`, composed on every `Avatar` inside the spine | The stream's state and every transition; the allowlisted `show()`. `Mixins.Broadcast`, `MixinApi.isBroadcastHost`. |
| `StandbyReason` · `LowerThirdPanel` | `lib/livestream/` (`@internal` value objects) | A reason row and a panel row, validated. |
| `standby-reason` · `lower-third-panel` · `stream-token` | `DocumentKinds` | Reasons and panels are platform-pack rows (`/platform/standby-reasons/<key>`, `/platform/lower-third-panels/<key>`); tokens are runtime-written (`/platform/stream-tokens/<sha256>`, `onVanish: keep`). |
| `StreamApi` ↔ `StreamLogic` | `api/stream.ts`, `platform/idea/api/StreamLogic.ts` | Tokens (`issueToken`/`resolveToken`), the away grammar (`interpretAway`, `parseDuration`), the rows (`standbyReasons`, `lowerThirdPanels`), per-streamer channel reads, the streamer's numbers. |
| `Events.StreamShowChanged` · `Events.PlayerFirstArrival` | `lib/events.ts` | `{ playerId, show }` on every change; a first arrival for the ticker. |
| `BroadcastFeed` | `backend/BroadcastFeed.ts` | Routes each streamer's show to the sockets opened with THEIR token; adds the world facts; remembers control sockets. |
| token doors | `WebSocketService` + `Backend` + `Application` | `?broadcast=<token>` (overlay) and `?control=<token>` (pblive); neither becomes an `Interactive`. |
| `handleControlCommand` | `backend/inbound/command.ts` | The control principal's one verb. |
| `privacy:private` | `MessageApi.Tags.Privacy.Private` | The server's word that a frame is private. |
| `stream` card | `Cards.ts` (client source) + client `StreamControls` | The controls; body is the `stream.state` client-state key. |

## The stream on the streamer's body

`BroadcastMixin` fields, all persistent (they ride the Avatar's
`holder_snapshots` record): `streamPhase` (`offline · starting · live ·
standby · ending`), `streamReason` (the reason row as resolved at `stream
away` — stored whole, so a restart needs no read to show it),
`streamStartsAt` / `streamAwayUntil` / `streamEndsAt` (epoch ms),
`streamTopic`, `streamAgenda` + `streamAgendaIndex`, `streamChannels`
(`{ twitch?, youtube?, kick? }`), `streamTokenHashes` (the live token per
scope). Transient: the outro timer, the last verb, the rotation's
pin/skip, the controls card's extras.

**On air** = `starting`, `live` or `ending` — the phases redaction covers.
Every transition refuses BY NAME on an illegal move (`not-streaming`,
`ending`, `already-on-air`, `no-agenda`, `agenda-done`, `unknown-panel`);
the controller renders the refusal. ⭐ `stream away` is legal from
`offline` too, so the shipped `stream away 15m` still means what it meant.

⏱ **Wall time, deliberately.** Deadlines are epoch ms and the outro is
`ScheduleApi` milliseconds (`stream.outroMs`, default 60 s): the stream is
interface tier and its minutes are the audience's. ⭐ **Only `ending →
offline` happens on a timer.** An Away or Starting countdown that runs out
changes nothing — only the streamer returns the desktop to the air (fail
closed). `rearmBroadcast()` runs at every body's `onCreate` (after
materialize for the body of record): an overdue outro goes offline at
once, a running one is re-booked, nothing else moves.

Every change emits `StreamShowChanged`, pushes `stream.state` to the
streamer's own client, and saves the body (the overlay must find the
state after a crash, and the autosave is minutes away). A show-relevant
client fact (`stream.foreground`, `cockpit.mode`, `cockpit.layout`)
re-emits while a stream is up; the VERB of each typed command is noted at
parse time (`CommandGiver`, never a forced command, never the arguments).

## The show — an allowlist

`show()` builds the wire `StreamShowCore` field by field: phase, reason,
the three timers, topic, agenda (`{ title, done }`) and `current`, the
client facts (`foreground`, `mode`, `layout`, `lastVerb`) while a stream
is up, and the rotation's pin/skip. The feed adds `playersOnline`
(`SocialApi.online()`), `build` (`SAXONBERG_BUILD_SHA` +
`SAXONBERG_BUILD_AT`, stamped into `.env` by `deploy/dev/update.sh`; unset
→ `null`, and the overlay hides the line) and the `panels` rows (listed
once per socket at connect). **Nothing of another player, no frame body,
no card content, no argument of any command.** The streamer-only numbers
(viewer counts, uptime, chat rate, who is online) go to `stream.state`
via `loadControls`, never to the feed.

## Tokens and the two doors

`stream token` mints 32 random bytes, shows them once, and stores only the
sha256 at `/platform/stream-tokens/<hex>` (`{ playerId, scope, issuedAt }`)
through the narrow `DocumentApi.saveStreamToken` transport (gated to
`StreamLogic`); issuing again revokes the previous one (rotation).
`stream token --control` does the same for the `control` scope.

- `?broadcast=<token>` — hash, read the path, require `scope: overlay` →
  principal `broadcast:<playerId>`; the feed sends the resident streamer's
  snapshot at once, or **nothing** until they load (never a synthesized
  phase). The env `BROADCAST_TOKEN` still opens the founder's feed during
  the changeover — the first id in `STREAMER_PLAYER_IDS` — and is refused
  loudly when that list is empty.
- `?control=<token>` — `scope: control` → principal `control:<playerId>`.
  Its only accepted message is a `command` whose line parses to ONE
  command with the verb literally `stream` (no `verb@source`, no emote),
  never `stream token`, never through an alias named `stream`; the
  streamer's body must be online; it runs as them through the ordinary
  dispatcher (`requiresStreamer` still gates) and every answer is echoed
  as `control-result` so the Stream Deck lights ✓ or ⚠.

A bad, revoked or wrong-scope token is a 401 with a warning — never a
fallback to a shared feed.

## Chat on the overlay

Each streamer's own channels — `stream channel <service> <handle>`, else
for the founder the `OVERLAY_*` env — are read under a per-streamer relay
sentinel (`overlay:<playerId>`), opened on that streamer's first overlay
socket and closed on their last. A relayed line reaches the overlays of
the streamer whose channel it is and no other. YouTube is found under the
streamer's own login when they have stepped up (see
[streaming.md](./streaming.md)), one quota unit instead of a search.

## Nobody is broadcast without being told

- **Redaction.** Tells, DMs, replies, hails, posts on a player-made
  channel and group handles carry `privacy:private`; a terminal card on a
  player-made channel or a group handle is flagged `private`. The
  streamer's client renders such a frame as *[private message]* and such a
  card as a husk **only** on those stamps and **only** while its own
  `stream.state` phase is on air. Standby and offline show everything —
  which is what a privacy standby is for. The record layer keeps tags, so
  a backfilled line redacts too.
- **The on-air marker.** `look`'s presence line and the person chip say
  `broadcasting`; the room says *"A broadcast reaches this place."* when
  anyone in it is on air. A statement, not a gate.

## Authorization — the streamer axis

The verb is afforded by `BroadcastMixin` (every played body) and gated by
`requiresStreamer` — `AccessApi.isStreamer`, membership of the
`streamers` managed group, seeded from `STREAMER_PLAYER_IDS`. A second
streamer is `group add streamers <player>`, a token, and their own Rodney
browser source: no code.

## The `stream` verb

`start [in <dur>]` · `live` · `away <reason> [<dur>]` / `away <dur>` ·
`back` · `end` · `off` · `segment <title>` · `agenda <a, b, …>` / `agenda
clear` · `next` · `channel <twitch|youtube|kick> [<handle>]` · `token
[--control]`, one controller per subcommand over `StreamControllerBase`
(controller-liquidity D13, at the merge with broadcast-ready). The lower
third is steered by its own verb, **`stream-rotation pin|skip <panel>` /
`stream-rotation clear`** — a non-diegetic compound name, because a
command is `verb subcommand target`, never deeper (controller-liquidity
W16; it was `stream rotation …`). `BroadcastMixin` affords both, and
both are gated `requiresStreamer`. Bare
`stream` prints the phase and opens the controls card (`opens_card:
stream`). Shipped reasons: `bathroom` 5m, `food` 15m, `water` 3m,
`privacy` (no timer, ever), `technical` (no timer), `generic` (what a
bare duration means).

## The client and the overlay

The client (W4/W5): the `stream` card (`StreamControls`), redaction, type
at 140% on air, the relay chat hidden on air, the foreground report
(`stream.foreground` on visibility/focus), `?from=stream` carried onto the
socket, *Link YouTube* in settings, and pop-out — see
[client-shell.md](./client-shell.md). Rodney reads the feed, maps phases
to OBS scenes by its own rules table, holds its scene when the feed
drops, and renders the panel rows (`packages/rodney/README.md`,
`docs/obs-setup.md`).

## Cross-references

- [access.md](./access.md) — the `isStreamer` predicate, `streamers`
  group, `requiresStreamer` validator.
- [connection.md](./connection.md) — the WS upgrade path the broadcast
  principal short-circuits.
- [comms.md](./comms.md) — the in-world `broadcast` ESP verb (the
  distinct concept this subsystem is *not*).
- [response-envelope.md](./response-envelope.md) — the `Envelope` wire
  union `StreamStateEnvelope`, `StreamEventEnvelope` and
  `ControlResultEnvelope` join.
- [streaming.md](./streaming.md) — the relay transports and the YouTube
  step-up the overlay's chat reads through.

## History

- **Phase 1 (broadcast feed)** — `StreamState` + `StreamStateEnvelope`
  + `?broadcast=` auth + `BroadcastFeed` + `requiresStreamer`/`streamers`
  + the `stream` verb. saxonberg `feature/broadcast-feed` (commit
  `532725ce`, MR !72). The overlay side shipped in `pbox-stream`
  `feature/phase-1-broadcast`.
- **broadcast-ready (2026-11)** — the stream moves onto the streamer's
  body (`BroadcastMixin`, five phases, reasons and panels as rows,
  durable), per-streamer tokens and feeds, the control door for pblive,
  redaction and the on-air marker, the YouTube step-up, the controls card
  and pop-out, Rodney on the real feed. `StreamState` retired.
  saxonberg `build/rodney`, MR !556 (`origin/master...build/rodney`);
  pblive `!1` and pblive-streamdeck `!1` in their own repos. What is
  left lives on broadcast-ready-slate.
  Drive record: the wire drive `packages/wire/tests/broadcast-ready.dirty.wire.test.ts`
  14/14 on a standing world, the browser walk `e2e/tests/broadcast-ready-walk.spec.ts`
  2/2. The drive found two defects — the agenda split on `;` (the shell's
  command separator; it splits on commas now) and an overlay snapshot that
  waited on the panel listing (it is sent at once now). Unproven on the wire,
  left to the owner's rehearsal: the Elgato cut, real Twitch/YouTube chat,
  OBS scene switching, a real process restart.
