# Livestream

The livestream subsystem is the server half of the **Rodney** overlay
(`packages/rodney`) and of a streamer's own controls: **the stream a
played body is running** (`BroadcastMixin`), the **program** it runs (a
document at `/platform/programs/<name>`), the `rodney` and `program` verbs,
the per-streamer **broadcast feed** that pushes an allowlisted show to
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
> also why the mixin is `BroadcastMixin` (the stream a body carries).

## The pieces

| Piece | Where | Role |
|---|---|---|
| `BroadcastMixin` | `lib/livestream/Broadcast.ts`, composed on every `Avatar` inside the spine | The stream's state and every transition; the allowlisted `show()`. `Mixins.Broadcast`, `MixinApi.isBroadcastHost`. |
| `StandbyReason` · `LowerThirdPanel` | `lib/livestream/` (`@internal` value objects) | A reason row and a panel row, validated. |
| `standby-reason` · `lower-third-panel` · `stream-token` | kind declarations (`/platform/kinds/<kind>`) | Reasons and panels are platform-pack rows (`/platform/standby-reasons/<key>`, `/platform/lower-third-panels/<key>`); tokens are runtime-written (`/platform/stream-tokens/<sha256>`, `onVanish: keep`). |
| `StreamApi` ↔ `StreamLogic` | `api/stream.ts`, `platform/idea/api/StreamLogic.ts` | Tokens (`issueToken`/`resolveToken`), the away grammar (`interpretAway`, `parseDuration`), the rows (`standbyReasons`, `lowerThirdPanels`), the program statics (`createProgram`, `amendProgram`, `attachFeed`, `detachFeed`, `runProgram`, `programGate`, `programListing`, `frontListing`, `programs()`), the streamer's numbers. |
| `program` kind · `Program` · `ProgramCatalogue` | `content/kinds/program.yaml`, `lib/livestream/Program.ts`, `platform/idea/ProgramCatalogue.ts` | A broadcast's standing record at `/platform/programs/<name>`: title, owner (a titled path) + `ownerName`, a one-line `schedule`, proven `feeds`, the `runner`. Written only by `StreamLogic` through `DocumentApi.saveProgram` (filing transport `program`). |
| `requiresProgram` | `lib/command/validators/requiresProgram.ts` | `rodney`'s gate: the giver's stream runs a program and the giver still holds its pen. |
| `FrontRoutes` | `backend/FrontRoutes.ts` | Anonymous `GET /api/front`: the named program (`?watch=&carrier=`), else the featured one (`stream.featuredProgram`). |
| `BroadcastApp` · `BroadcastDesk` · `OnAirSign` | `platform/idea/BroadcastApp.ts`, `platform/thing/` | What affords `rodney` + `program`: the app on a phone you carry ([aether-devices.md](./aether-devices.md)), or a desk you sit at (`peers`). The sign's light is derived. |
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
`streamTopic`, `streamAgenda` + `streamAgendaIndex`, `streamProgram` (the
program whose chair this stream holds — `program run`; cleared, quietly, if
the program is gone or the pen lost), `streamTokenHashes` (the live token
per scope). The body no longer carries channels: they are the program's. Transient: the outro timer, the last verb, the rotation's
pin/skip, the controls card's extras.

**On air** = `starting`, `live` or `ending` — the phases redaction covers.
Every transition refuses BY NAME on an illegal move (`not-streaming`,
`ending`, `already-on-air`, `no-agenda`, `agenda-done`, `unknown-panel`);
the controller renders the refusal. ⭐ `rodney away` is legal from
`offline` too.

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

`rodney token` mints 32 random bytes, shows them once, and stores only the
sha256 at `/platform/stream-tokens/<hex>` (`{ playerId, scope, issuedAt }`)
through the narrow `DocumentApi.saveStreamToken` transport (gated to
`StreamLogic`); issuing again revokes the previous one (rotation).
`rodney token --control` does the same for the `control` scope.

- `?broadcast=<token>` — hash, read the path, require `scope: overlay` →
  principal `broadcast:<playerId>`; the feed sends the resident streamer's
  snapshot at once, or **nothing** until they load (never a synthesized
  phase). There is no env door: `BROADCAST_TOKEN` and the other four
  stream env names (`STREAMER_PLAYER_IDS`, `OVERLAY_TWITCH_LOGIN`,
  `OVERLAY_YOUTUBE_CHANNEL`, `OVERLAY_KICK_CHANNEL`) retired with stream
  arrival, and boot warns if one is still set.
- `?control=<token>` — `scope: control` → principal `control:<playerId>`.
  Its only accepted message is a `command` whose line parses to ONE
  command with the verb literally `rodney` (no `verb@source`, no emote),
  never `rodney token`, never through an alias named `rodney`; the
  streamer's body must be online; it runs as them through the ordinary
  dispatcher (`requiresProgram` still gates) and every answer is echoed
  as `control-result` so the Stream Deck lights ✓ or ⚠. A deck still
  sending `stream …` is refused `rodney-only`; an alias named `rodney`,
  `alias-shadows-rodney`.

A bad, revoked or wrong-scope token is a 401 with a warning — never a
fallback to a shared feed.

## Chat on the overlay

The channels are **the program's feeds** — the program the streamer's
stream runs (`program run`) — read under a per-streamer relay
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

## The program — who may run a stream

⭐ **Nobody is *the* streamer; the unit is the program.** A program is a
document at `/platform/programs/<name>` (kind `program`): a short name
(first come, never changes), a title, an **owner** — a titled path whose
pen edits it (a player's own home, or an institution's branch with
`--owner`) — a one-line schedule, and its feeds. **The pen is the document
store's own write gate, `DocumentApi.mayWrite`** (own `/home/<key>/`
first, else `AccessApi.canAtPath`) asked of `<owner>/programs/<name>`.

A **feed is proven at attach**, never typed: `program channel <name>
<carrier>` reads the attacher's OWN linked account — `TwitchProfile.login`,
`KickProfile.slug`, or `GoogleProfile.youtubeChannelId` after *Link
YouTube* — and records who proved it. A program with no feed is still a
program.

`program run <name>` puts the giver's stream in its chair (`runner`;
`streamProgram` on the body). **`requiresProgram`** then gates `rodney`:
the stream runs a program and the giver still holds its pen. A second
broadcaster is a second program — no group, no env, no code.

## The `rodney` and `program` verbs

`rodney` drives the overlay (named for what it drives — stream arrival
D22; it was `stream`): `start [in <dur>]` · `live` · `away <reason>
[<dur>]` / `away <dur>` · `back` · `end` · `off` · `segment <title>` ·
`agenda <a, b, …>` / `agenda clear` · `next` · `token [--control]` ·
`pin <panel>` · `skip <panel>` · `unpin` (the lower third, flat — the
`stream-rotation` verb is gone). One controller per subcommand. Bare
`rodney` prints the phase and opens the controls card. Shipped reasons:
`bathroom` 5m, `food` 15m, `water` 3m, `privacy` (no timer, ever),
`technical` (no timer), `generic` (what a bare duration means).

`program`: `create <name> [--title …] [--owner <path>]` · `retitle` ·
`schedule <name> <text>` / `--clear` · `channel <name> <carrier>` /
`--clear` · `run <name>` · `about <name>`. Bare `program` shows the
program your stream runs.

⭐ **Neither verb is on the body.** They are afforded by the **Broadcast
app** on a device you bear (`apps install broadcast on phone` — an app's
verbs are lent to whoever carries its device, see
[aether-devices.md](./aether-devices.md)), or by a **broadcast desk**
(`peers`). Hallett's booth, off the University Avenue crossing in
Terminus, has the desk and an **on-air sign** whose light is DERIVED —
lit while anyone in the booth is a broadcast host on air — never switched
(`OnAirSign` overrides `getEmittedFlux` and `getLong`). The testbed hut has
both.

## The front page's show

`stream.featuredProgram` (Tier B setting, `config stream.featuredProgram
<name>`) names the program the anonymous `GET /api/front` answers with,
unless a link named one (`?watch=<name>&carrier=<c>`). The answer carries
the title, owner name, live flag, next show and the PROVEN feeds only;
nothing is shown when no program is named or featured.

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

- [document-store.md](./document-store.md) — the `program` kind and its
  filing transport; `DocumentApi.mayWrite`, the pen.
- [aether-devices.md](./aether-devices.md) — the Broadcast app, and how an
  installed app lends its verbs.
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
  Drive record: the wire drive `packages/wire/tests/broadcast-ready.wire.test.ts`
  14/14 on a standing world, the browser walk `e2e/tests/broadcast-ready-walk.spec.ts`
  2/2. The drive found two defects — the agenda split on `;` (the shell's
  command separator; it splits on commas now) and an overlay snapshot that
  waited on the panel listing (it is sent at once now). Unproven on the wire,
  left to the owner's rehearsal: the Elgato cut, real Twitch/YouTube chat,
  OBS scene switching, a real process restart.
- **Stream arrival (2026-11-24, MR !655)** — the unit became the
  **program** (a document, owned through the pen, feeds proven at attach);
  `stream` → **`rodney`** + **`program`**; `requiresStreamer`, the
  `streamers` group, `stream-rotation`, `stream channel` and the five env
  names retired; the verbs moved off the body onto the Broadcast app and
  the broadcast desk; Hallett's booth and its derived on-air sign;
  `stream.featuredProgram` + `GET /api/front`. Drive:
  `packages/wire/tests/stream-arrival.wire.test.ts`.
