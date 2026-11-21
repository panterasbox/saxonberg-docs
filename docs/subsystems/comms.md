# Comms

The communication substrate — how beings talk to each other, near
and far. Comms is the integrating spine over two **honestly distinct
transports**: an **acoustic** channel (you *hear* it — `say`,
`whisper`, `shout`) and an **implant / ESP** channel (you *know* it,
a thought arriving — `dm`). It owns the speech verb family
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

> ⭐ **The aether's rule — [arcane-science.md](../arcane-science.md) § *The second postulate***
> (2026-10-29): information only, at once across any distance in Coordinated Metaversal Time, by the
> field where you stand; every message carries its true sender.

The load-bearing decision: communication travels one of two ways, and
they are *not* the same medium.

| | **Acoustic** | **Implant (ESP)** |
|---|---|---|
| comm types | `say`, `whisper`, `shout` | `dm` (+ chat channels, remote emote) |
| perceived via | the senses — you *hear* it | direct cognition — you *know* it |
| modality stamp | `meta.modality: 'hearing'` | `meta.modality: 'verbal-esp'` |
| gated by | sound reach, hearing, language | attribution only; no sensory gate |
| reach | physical space (distance, walls) | membership / addressing (distance-free) |
| privacy | public, overhearable | private, addressed |
| carrier mixin | `VocalMixin` | `CommsMixin` (an app on `AetherMixin` attunement) |

Acoustic is local, atmospheric, overhearable, and (eventually)
language-bound; implant is distance-free, private, and "magic." The
two mixins compose independently onto a host — a character can be
Vocal without being Aether (mute, or pre-augmentation) and Aether
without Vocal (post-vocal-loss with an implant). See
[messaging.md § VocalMixin / AetherMixin](./messaging.md) for the
capability split and the Containable-wins audience rule both share.

Diegetically a tuner lets you read and write the aether field — a
**phone** read off its screen (every player's, from the starting kit), an
**implant** known straight to the mind (a guest's, or a headset worn). The
same Aether substrate carries DMs, chat channels, and the remote perception
of emotes. A local emote needs no tuner: the room carries it. Which tuner,
when it works, apps, the battery and held messages:
[aether-devices.md](./aether-devices.md).

⚠ **Every Avatar, never any NPC.** `CommsMixin` is composed on `Avatar`
only; no NPC composes it or any tuner, so `dm` cannot reach an NPC and a
remote emote passes it by — ⛔ **and that is final**: distance comms are
player-to-player only — you talk to an NPC with `say`, in person (the
owner, 2026-11-18; aether-devices-slate
§ 2). An NPC does perceive every emote made in its room.

### Comms is the person's; Messages is the app it looks for

Three mixins, three facts (aether-devices D2): `AetherMixin` = *can
perceive the aether when tuned* (the `verbal-esp` / `emotive-esp`
modalities reception-gating rides); `AetherHostMixin` = *hosts apps*;
`CommsMixin` = *this person talks at a distance, and things wait for them*
— composed on the person (`Avatar`). `tell` sends **as the person**: the
self-echo frame, the mention resolver and the speaker name are the
person's own.

Before anything goes, the person asks `sendOrHold(commandText)`: is there
a host that tunes them right now (`aetherHostNow` — tuned by nature, or the
first augment they bear that nothing stops), and does it run **Messages**
(`/platform/idea/app/messages`)? If not, the typed line is held and the
refusal names the fix; it re-runs when it can. A frame arriving over the
aether that cannot be taken now is held the same way. See
[aether-devices.md § Held, never silent](./aether-devices.md).

## The verb surface

Six verbs ship across the two transports. Three acoustic (gated by
`VocalMixin`), one aether directed-message (asked of the person's
`sendOrHold`), plus the two aether pronoun/broadcast verbs that ride
alongside.

| Verb | YAML | Controller | Transport | Mixin method |
|---|---|---|---|---|
| `say` (`'`) | `cmd/say.yaml` | `SayController` | acoustic | `VocalMixin.say` |
| `whisper` | `cmd/whisper.yaml` | `WhisperController` | acoustic | `VocalMixin.whisper` |
| `shout` | `cmd/shout.yaml` | `ShoutController` | acoustic | `VocalMixin.shout` |
| `dm` | `cmd/dm.yaml` | `DmController` | aether | `CommsMixin.tell` (the person's) |
| `reply` | `cmd/reply.yaml` | `ReplyController` | aether | cohort replay of `CommsMixin.tell` |
| `broadcast` | `cmd/broadcast.yaml` | `BroadcastController` | aether | composes its own Scene |

The acoustic verbs are contributed via
`VocalMixin.commandContributions.self`
(`['say.yaml', 'whisper.yaml', 'shout.yaml']`); the aether verbs via
`CommsMixin.commandContributions.self`
(`['dm.yaml', 'reply.yaml', 'broadcast.yaml', 'chat.yaml']`) — the
person's own `self` bucket, since comms lives on the person. The
`chat.yaml` contribution is the channel surface — its substrate lives in
[chat](./chat.md), not here.

Each controller is a thin shell. The acoustic ones narrow on
`MixinApi.isVocal` and report `mixin-missing` + "You cannot speak." The
`dm` / `reply` controllers (and a `chat` post) ask the person's
`sendOrHold` and render a refusal (`aether-<reason>`) or call `tell`. No
validator stands in front of them: `requiresVerbalESP` would refuse
before the hold could happen, so it stays only on `forum` and
`broadcast`, where it says the person's own fix.

### Acoustic — say / whisper / shout

One primitive — **vocalize at volume V** — with three presets. The
shared `vocalEmit` helper in `VocalMixin` composes the body and fires
the Scene; the verb word ("say" / "whisper" / "shout") drives the
prose and the dB + topic come from the calling method.

- **`say`** — normal volume (60 dB), room reach, undirected by
  default. Topic `speech.vocal`.
- **`whisper`** — quiet (30 dB), short reach *as stamped* (see below). Topic
  `speech.quiet`. `whisper.yaml` makes `target` a **required**
  positional arg — whisper is implicitly directed (you whisper *to*
  someone).
- **`shout`** — loud (90 dB), multi-room reach *as stamped* (see below). Topic
  `speech.vocal`.

⚠ **The dB is a stamp; the reach is the room.** `vocalEmit` fans the
peers frame through `Scene.toPeers` — the speaker's one room — for all
three verbs, and nothing downstream reads `acousticDb` for speech
(`Scene.toAudible` + `AudienceGather` exist, but only `Audible.emit` —
whistles, bells — rides them; `senses.md § Deferred`: *existing comms
ship unchanged*). So a whisper reaches every peer in the room **with
the words** (*"X whispers to Y, <the words>"*, never a redacted
*"whispers something"*), and a shout does not leave the room. The comms
slate designed both the redaction and the cross-room shout; neither
shipped.

All three stamp `meta.modality: 'hearing'` for sensorium gating, and
emit to peers via the Containable-wins rule (a Containable speaker
addresses its environment's peers; a pure-Container speaker — a
haunted room — addresses its own contents). See
[messaging.md](./messaging.md) for that rule's detail.

### Implant — dm

`dm` is the directed-message verb (its `tell` alias was deleted by controllers-are-io's one view, one word). `dm.yaml` takes a
`type: objects` target with `cardinality: { min: 1, max: 10 }` at
`scope: online`, plus a greedy `message`. `DmController` asks the
person's `sendOrHold` (held + a refusal naming the fix, or go) and
delegates to `CommsMixin.tell`, which stamps `meta.modality:
'verbal-esp'` and fires `speech.comms` *from the person*.

Three cardinality cases inside the controller (`comms` = the person):

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
lives on the **person** (`CommsMixin`), stamped automatically by `tell` —
runtime-only, never persisted (so a `reply` held across a restart finds
nobody and says so). On send, `tell` records the recipient's inbound
cohort on the recipient (`MixinApi.isComms(t)` — a person composes it).
`reply` / `dm .` read the cohort for reply-all. The `dm <ad-hoc-handle>
...` post-to-known-channel shape is **not** in v1; posting to an ad-hoc
cohort goes through `chat <handle> ...`.

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
object`, `scope: online`, `field: target`) with `requires: class:Agent`
(+ `onExcess: prompt`). `whisper` is always-directed, so it takes `target`
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

## ⭐ The privacy stamp (broadcast-ready D8)

Every tell / DM / reply frame — self and target — carries the
framework tag `privacy:private` (`MessageApi.Tags.Privacy.Private`), as
does a hail. It is the server's word that the frame is private: a
streamer's client redacts exactly these while on air
([livestream.md](./livestream.md)), and the record layer keeps tags, so a
backfilled tell redacts too.

## Relation to messaging and chat

Comms is a **source** feeding the [messaging](./messaging.md)
pipeline, not a parallel one. Every verb composes through
`MessageApi.scene` and goes out the single `onMessage`
delivery chokepoint:

- **MML / Scene composer** — `messaging.md` owns the
  actor/target/peers audience taxonomy, MML body composition, the
  markdown→MML pipeline (so `**bold**`, `@name` mentions, and
  `[label](mudcmd:…)` work in speech bodies), and `commandId` /
  `causingCommandId` auto-attribution. Comms supplies the verb prose
  and the topic/modality/meta stamps; the composer does the rest.
- **Reception gating** — the `'hearing'` vs `'verbal-esp'` modality
  stamp routes through `SensorMixin.filterMessage`. For a person the
  aether frames meet the HOLD first (`Avatar.filterMessage` →
  `CommsMixin.holdIfUnreachable`): what cannot be taken now waits under
  *"While you were away…"* rather than dropping. Acoustic frames
  additionally feed the senses substrate's reach walk via `acousticDb`.

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
directed path, and the `dm` implant primitive. The
comms slate retains the rest of the
design space. Designed but **not yet built**:

- ⭐⭐⭐ **Carriers other than the implant** — ✅ the first is BUILT: the
  **phone**, a portable device conferred by carrying
  ([aether-devices.md](./aether-devices.md)). The rest of the ladder below
  is still design. The ESP channel's modality
  (`verbal-esp`) never changes; only the **coupling** does, and any
  carrier reaches any other. Three sources, not one: the implant, an
  intrinsic species trait, and a **portable device**, conferred the way
  `augmentation.md` confers any mixin. ⭐ The ladder's axis is
  **portability, not sophistication** — prehistory a *place* you go to,
  medieval an *object* you carry (the sympathetic needle), industrial a
  *device* you operate (an aether-radio, pre-transistor but off the EM
  band), modern an *organ*. ⛔⛔ **Communication is never a regulated act**
  — an epoch covenant restricts the instrument bound to a *regulated*
  act and never asks what you are carrying, which is what keeps
  *never cut players off from their peers* structural rather than an
  exception. What epoch may flavour is privacy, latency, length, cost
  and attribution — *strange, not silent*. Design:
  content-declaration-slate § 5.

  > ⚠⚠ **AMENDED 2026-10-09 — one clause, and the invariant is
  > untouched.** *"Never asks what you are carrying"* was written against a
  > **comms-specific** rule and did not anticipate a **general-tech** rule
  > reaching a comms instrument *incidentally*: a monastery that forbids
  > modern machinery is regulating machinery, and the implant falls in that
  > class whether or not comms was the point.
  >
  > ⭐⭐⭐ **So a covenant MAY regulate modern tech as a class. What it
  > may never do is leave you with no instrument at all** — and the
  > portability ladder above is what guarantees it cannot: **every band
  > has an instrument.** You hand over the modern *organ* and you are
  > holding the medieval *needle*. ***Strange, not silent* — satisfied
  > exactly.** The invariant does not move; only its mechanism does, from
  > *"we never ask"* to *"every band has an instrument."*
  >
  > ⚠ And the enforcement rung is bounded: ⭐⭐ **the `wall` rung can
  > refuse an object at a boundary and cannot perform surgery**, so a
  > no-implant rule is `camera`/`witness`/`norm` strength, never `wall`.
  > ⛔ Also note that **the AETHER itself is unregulable — it is a physical
  > background field**, and this doc's framing of it as *"implant / ESP"*
  > names one receiver of two: a carried **phone** is the other.
  > → content-declaration-slate § 3.8.

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
  `onMessage` delivery chokepoint, modality stamping,
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
