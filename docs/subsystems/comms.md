# Comms

The communication substrate — how beings talk to each other, near
and far. Comms is the integrating spine over two **honestly distinct
transports**: an **acoustic** channel (you *hear* it — `say`,
`whisper`, `shout`) and an **implant / ESP** channel (you *know* it,
a thought arriving — `dm` / `tell`). It owns the speech verb family
and the directed-message primitive; it hands the rest off — acoustic
reach to the senses substrate, the channel model to
[chat](./chat.md), and frame composition / delivery to
[messaging](./messaging.md).

This doc covers what Wave 1 shipped: the verb surface, the
`meta.acousticDb` stamping, the `say --to` directed-speech path, and
the whisper-vs-tell transport split. The channel half (named, lived-in
chat channels) graduated into [chat](./chat.md); the unbuilt waves
(dynamic-reach shout, regional channels, the conversation primitive,
implant security) are flagged in **Deferred** below.

## Two transports — acoustic vs implant

The load-bearing decision: communication travels one of two ways, and
they are *not* the same medium.

| | **Acoustic** | **Implant (ESP)** |
|---|---|---|
| comm types | `say`, `whisper`, `shout` | `dm` / `tell` (+ chat channels, remote emote) |
| perceived via | the senses — you *hear* it | direct cognition — you *know* it |
| modality stamp | `meta.modality: 'hearing'` | `meta.modality: 'verbal-esp'` |
| gated by | sound reach, hearing, language | attribution only; no sensory gate |
| reach | physical space (distance, walls) | membership / addressing (distance-free) |
| privacy | public, overhearable | private, addressed |
| carrier mixin | `VocalMixin` | `CommsMixin` (a hosted update on `AetherMixin` attunement) |

Acoustic is local, atmospheric, overhearable, and (eventually)
language-bound; implant is distance-free, private, and "magic." The
two mixins compose independently onto a host — a character can be
Vocal without being Aether (mute, or pre-augmentation) and Aether
without Vocal (post-vocal-loss with an implant). See
[messaging.md § VocalMixin / AetherMixin](./messaging.md) for the
capability split and the Containable-wins audience rule both share.

Diegetically the implant is **cybernetic in mechanism, ESP in
phenomenology** — a neural interface delivering coherent thoughts
willed into existence, bypassing the sense organs. The same Aether
substrate carries DMs, chat channels, and the remote perception of
emotes. The baseline AetherImplant is universal and always-on for
players (see [augmentation.md](./augmentation.md)), which is why
`dm` / chat are zero-friction.

### Comms is a hosted update; `AetherMixin` is the host

Transmission is no longer carried by `AetherMixin`. **Attunement**
(`AetherMixin`) = perceive the aether (the `verbal-esp` / `emotive-esp`
modalities reception-gating rides) + *host* capability updates. The
**comms capability** is `CommsMixin`, composed on a hosted update
`CommsUpdate` (`CommsMixin(AetherHostedMixin(Idea))` —
`lib/comms/Comms.ts` + `CommsUpdate.ts`) that plugs into an attunement
host. `tell` sends *on behalf of its host (the operator)*: it resolves
the operator via `getHost()` and routes the self-echo frame, the mention
resolver, and the emitted speaker name through the host (a comms update
is not itself a `Sensor` and has no name). The host/update hosting
relation, the three-base capability model, and the reachability scan
that finds the update live in [augmentation.md](./augmentation.md).

Net: **attunement = perceive + host; the comms update = transmit.** A
recipient who is attuned but lacks the comms update still *receives*
dms (the reception gate passes), they just can't *send*. The future
radio is the corporeal twin: an attuned `Thing` that **hosts** a comms
update (not a `Thing` that composes `CommsMixin`).

## The verb surface

Six verbs ship across the two transports. Three acoustic (gated by
`VocalMixin`), one implant directed-message (gated by `AetherMixin`),
plus the two Aether pronoun/broadcast verbs that ride alongside.

| Verb | YAML | Controller | Transport | Mixin method |
|---|---|---|---|---|
| `say` (`'`) | `cmd/say.yaml` | `SayController` | acoustic | `VocalMixin.say` |
| `whisper` | `cmd/whisper.yaml` | `WhisperController` | acoustic | `VocalMixin.whisper` |
| `shout` | `cmd/shout.yaml` | `ShoutController` | acoustic | `VocalMixin.shout` |
| `dm` / `tell` | `cmd/dm.yaml` | `DmController` | implant | `CommsMixin.tell` (on the hosted update) |
| `reply` | `cmd/reply.yaml` | `ReplyController` | implant | cohort replay of `CommsMixin.tell` |
| `broadcast` | `cmd/broadcast.yaml` | `BroadcastController` | implant | composes its own Scene from the host |

The acoustic verbs are contributed via
`VocalMixin.commandContributions.self`
(`['say.yaml', 'whisper.yaml', 'shout.yaml']`); the implant verbs via
`CommsMixin.commandContributions.self`
(`['dm.yaml', 'reply.yaml', 'broadcast.yaml', 'chat.yaml']`) — note these
moved off `AetherMixin` when comms became a hosted update. They reach the
host's command-source walk through the **hosted-update self-seeding**
(`CommandApi.collectHostedUpdateDefs` + `applyHostedUpdateDelta`), with
the comms update as `commandSource` (exactly one source — see the
[augmentation.md](./augmentation.md) contribution-walk generalization).
The `chat.yaml` contribution is the channel surface — its substrate
lives in [chat](./chat.md), not here.

Each controller is a thin shell. The acoustic ones narrow on
`MixinApi.isVocal` and report `mixin-missing` + "You cannot speak." The
`dm` / `reply` controllers resolve the operator's hosted comms update —
preferring `context.commandSource` (the update that afforded the verb),
else the MQL `reachable` pool filtered on `isComms` — then invoke
`tell` on it; if no comms update is found (attuned but update-less) they
fire the `mixin-missing` "no way to send a thought" refusal. **This
absent-comms gate lives in the controller, not the validator**:
`requiresVerbalESP` is the attunement early-catch (perceive), but
attunement alone doesn't guarantee a comms update (transmit) — see
*Reception gating* below.

### Acoustic — say / whisper / shout

One primitive — **vocalize at volume V** — with three presets. The
shared `vocalEmit` helper in `VocalMixin` composes the body and fires
the Scene; the verb word ("say" / "whisper" / "shout") drives the
prose and the dB + topic come from the calling method.

- **`say`** — normal volume (60 dB), room reach, undirected by
  default. Topic `speech.vocal`.
- **`whisper`** — quiet (30 dB), short reach. Topic
  `speech.quiet`. `whisper.yaml` makes `target` a **required**
  positional arg — whisper is implicitly directed (you whisper *to*
  someone).
- **`shout`** — loud (90 dB), multi-room reach. Topic
  `speech.vocal`.

All three stamp `meta.modality: 'hearing'` for sensorium gating, and
emit to peers via the Containable-wins rule (a Containable speaker
addresses its environment's peers; a pure-Container speaker — a
haunted room — addresses its own contents). See
[messaging.md](./messaging.md) for that rule's detail.

### Implant — dm / tell

`dm` (alias `tell`) is the directed-message verb. `dm.yaml` takes a
`type: objects` target with `cardinality: { min: 1, max: 10 }` at
`scope: online`, plus a greedy `message`. It carries the
`requiresVerbalESP` validator so an unattuned host is turned away
before dispatch. `DmController` resolves the hosted comms update and
delegates to `CommsMixin.tell`, which stamps `meta.modality:
'verbal-esp'` and fires `speech.comms` *from the operator*.

Three cardinality cases inside the controller (`comms` = the resolved
hosted comms update):

1. **Single target** → `comms.tell(target, message)` — the classic
   1:1 rendering ("X → Y: msg" to self, "X → you: msg" to target).
2. **2–10 targets** → opens an **ad-hoc Channel** via
   `ChatApi.openAdHoc` so subsequent `chat <handle> ...` posts route
   to the same cohort, then `comms.tell(targets, message, {
   channelId })`. The ad-hoc channel half is documented in
   [chat.md § Three channel kinds](./chat.md).
3. **> 10 targets** → refused with a `controller-rejected`
   (`recipient-cap-exceeded`) note and a self-frame steering the
   player to `chat make <name>`. Groups that size belong on a real
   channel with membership / subscription / moderation.

DM **cohort state** (`getLastInboundCohort` / `getLastOutboundCohort`)
lives on the **comms update** (`CommsMixin`), stamped automatically by
`tell` — runtime-only, never persisted, dies with the host. On send,
`tell` records the recipient's inbound cohort on the *recipient's own*
comms update (found via the recipient's own MQL `reachable` pool,
null-guarded — an attuned
recipient with no comms update receives the dm but has nowhere to record
a cohort, which is fine: they can't reply anyway). `reply` / `dm .`
read the cohort off the operator's comms update for reply-all. The `dm
<ad-hoc-handle> ...` post-to-known-channel shape is **not** in v1;
posting to an ad-hoc cohort goes through `chat <handle> ...`.

## `meta.acousticDb` stamping

The single piece of physical-reach metadata comms contributes is
`meta.acousticDb` — the **source level** in decibels, stamped on every
acoustic frame for the sound-propagation walk's reach computation.
`VocalMixin` holds a flat per-verb table:

```ts
const DB = { whisper: 30, say: 60, shout: 90 } as const;
```

Reach is *not* computed here — that's the senses substrate's job.
Comms only declares how loud the source is; who actually hears it
(distance, walls, masking) is resolved downstream. The implant family
carries no `acousticDb` at all — implant reach is membership, not
space.

**v1 ceiling**: these are flat scalars — every speaker shouts at the
same volume regardless of size, condition, or skill. The vitals
substrate (deferred) is the natural seam to vary `acousticDb`
per-speaker (constitution / projection / fatigue); when it lands, the
constant lookup becomes a computed value.

## Directed speech — `say --to`

Directedness is an **option, not a default** — `say` is undirected
room chat ~95% of the time, so addressing opts in via `--to` rather
than guessing whether the first word is a name or the message.
`say.yaml` and `shout.yaml` carry a `to` option (`short: t`, `type:
object`, `scope: online`, `field: target`) with a `mustBeAgent`
validator. `whisper` is always-directed, so it takes `target`
positionally instead.

The general rule comms follows: **free-prose-tail verbs direct via
`--to`; structured-tail verbs direct positionally** (`say --to iffy
…` vs the emote `smile iffy`).

Directed speech is **public but addressed** — the room still hears,
and the target gets its own target-frame. `vocalEmit` renders three
audiences when a target is present:

- self — "You say to *Y*, …"
- peers — "*X* says to *Y*, …"
- target — "*X* says to you, …"

The target-frame is what marks an NPC as being spoken *to* — the seam
a future dialogue responder keys on. Without `--to`, only the self
and peers frames render ("You say, …" / "*X* says, …").

## Relation to messaging and chat

Comms is a **source** feeding the [messaging](./messaging.md)
pipeline, not a parallel one. Every verb composes through
`MessageApi.scene` and goes out the single `MessageApi.sendMessage`
delivery chokepoint:

- **MML / Scene composer** — `messaging.md` owns the
  actor/target/peers audience taxonomy, MML body composition, the
  markdown→MML pipeline (so `**bold**`, `@name` mentions, and
  `[label](mudcmd:…)` work in speech bodies), and `commandId` /
  `causingCommandId` auto-attribution. Comms supplies the verb prose
  and the topic/modality/meta stamps; the composer does the rest.
- **Reception gating** — the `'hearing'` vs `'verbal-esp'` modality
  stamp routes through `SensorMixin.filterMessage`, the same
  reception gate that drops implant frames for unattuned
  recipients. Acoustic frames additionally feed the senses
  substrate's reach walk via `acousticDb`. Note the
  **attunement-vs-comms split**: reception rides attunement (the
  `verbal-esp` sensorium gate, residual `AetherMixin`), so it stays
  green for any attuned actor; the *transmit* gate (do you have a comms
  update) is enforced in the `dm`/`reply` controllers, not the
  validator.

The **channel half** of the comms design — named, persistent,
membered chat channels — shipped under [chat](./chat.md), not here.
Chat consumes the `AetherMixin` `'verbal-esp'` transport and the group
substrate; comms owns the speech verbs and the directed-message
primitive that *mints* the ad-hoc channel chat then manages. The
boundary: comms = the verb-to-transport family + 1:1 / small-cohort
directed messages; chat = the lived-in channel model on top of the
implant transport.

## Deferred

Wave 1 shipped the acoustic say/whisper/shout family, the `say --to`
directed path, and the `dm`/`tell` implant primitive. The
comms slate retains the rest of the
design space. Designed but **not yet built**:

- **Dynamic-reach shout** — shout currently stamps a flat 90 dB. The
  designed extension makes output dB scale with a **voice-projection
  attribute** (and clarity degrade with distance via acoustic
  attenuation, see senses.md), with vitals (lung capacity / fatigue) throttling it.
  The `acousticDb` seam exists; the attribute does not.
- **Language gating** — acoustic comprehension (do you understand the
  words) routes to the language slate; the `'hearing'` frames carry no
  language metadata yet. Implant leans toward *encoded cognition*
  (you think in a language; the receiver needs it, or a translation
  implant) — also unbuilt.
- **Regional channels** — a channel scoped to a zone, bridging
  acoustic locality and implant networks. Open question in the slate.
- **The conversation primitive** — a first-class DM/group conversation
  object with its own identity and lifetime. v1 has the degenerate
  cases (1:1 `tell`, multi-target → ad-hoc Channel via chat) but no
  unified conversation entity.
- **Implant security** — attribution is the implant trust boundary
  ("this thought is *X*'s, not yours"); the high-end threat is
  spoofing (injected or impersonated thoughts). The baseline is
  hardened against casual jamming by design; the espionage/horror
  security layer is its own wave.
- **Async mail** and the **moderation control plane** (the
  expression-policy gate keying on channel identity) — adjacent
  futures.


> ⭐ **`<spoiler>` is admitted here** (`tags: 'spoiler'`), and it is the
> only literal markup a conversation carries. Chat is where a spoiler
> gets blurted — a channel hears "the boss is a mimic" long before
> anybody writes the article — and only the *appetite* half of the
> reveal model applies: a chat line has no authored capability level,
> so there is nothing to gate, only something for the client to fold.
> Everything else escapes, because `<link>`/`<mention>` are affordances
> that ISSUE COMMANDS and `<speech>`/`<name>` are identity claims the
> composer emits on the server's authority. See
> [wiki.md](./wiki.md) § the tag policy.

---

## Related

- [messaging.md](./messaging.md) — the Scene composer,
  `MessageApi.sendMessage` delivery chokepoint, modality stamping,
  `SensorMixin.filterMessage`, and the `VocalMixin` / `AetherMixin`
  capability split with the Containable-wins audience rule.
- [chat.md](./chat.md) — the channel model riding the implant
  transport: `Channel` Documents, the three channel kinds, the ad-hoc
  channel `dm` mints, `ChannelCatalogue`.
- [emotes.md](./emotes.md) — expression riding the Aether/ESP channel;
  `SoulMixin` grants local emote, remote emote needs both `SoulMixin`
  and `AetherMixin`.
- [augmentation.md](./augmentation.md) — the baseline AetherImplant
  that makes `'verbal-esp'` universal for players and gates
  `AetherMixin` active.
- [message-rendering.md](./message-rendering.md) — end-to-end render
  of the frames comms emits (server MML + client parse + theme
  cascade).
- comms slate — the full two-transport
  design and the deferred waves this doc's **Deferred** section
  summarizes.
