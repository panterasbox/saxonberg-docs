# Chat

Chat is the multi-party communication substrate riding on top of the
[messaging](./messaging.md) pipeline. Where `dm` is a directed message
to one (or a small cohort of) addressees, a chat **channel** is a
named, lived-in conversation surface — persistent or ephemeral, open
or membered, player-created or engine-seeded. The wire still flows
through `MessageApi.scene` / `MessageApi.sendMessage`; chat adds the
channel model, the audience-resolution layer, and the runtime history
ring that sit between "actor speaks into a channel" and the per-
recipient frames produced by the messaging substrate.

Diegetically every channel is a frequency on the baseline AetherImplant
(see [augmentation.md](./augmentation.md)). There is no IC/OOC overlay
and no `isIC` property; Help is a *mentor frequency*, Global is the
*common band*, a player-made channel is a *private frequency*. The
modality stamp on every chat frame is `'verbal-esp'`, so reception
gates through the same `SensorMixin.filterMessage` modality check that
governs `tell` — implant-less recipients miss chat without further
wiring.

Chat is distinct from `dm` (a single-cohort directed-speech verb that
mints one off the cuff frame and does not lay down a channel) and from
**MudlogApi** (`shell.diagnostic.*` — the game narrating to the player, not
agents conversing with one another). Game-event feeds belong on
`MudlogApi`; chat is for messages between actors.

## Three channel kinds

The v1 channel model is deliberately small. Three kinds, each with a
fixed origin story and lifetime:

- **player-created** — minted by an explicit `chat make <name>` verb.
  Persistent: stored as a `Channel` Document in the `channels`
  collection. Carries an explicit member roster (creator joins as
  `owner`); only members are eligible to post or receive.
- **open-join-standalone** — engine-seeded (`Help`, `Global`, `Chat`)
  or, in time, committee-created. Persistent. Has no member
  roster at all — eligibility is universal. Audience is gated only
  by per-player subscription.
- **ad-hoc** — minted on the fly when a `dm` is sent to more than one
  target. Runtime-only: lives in the `ChannelCatalogue`'s in-memory
  registry under a short generated handle and dies on disband or
  server restart. Subscription is implicit (members are everyone in
  the cohort).

The three kinds share the same `speech.channel` topic, the same
`meta.channelId` wire stamp, the same history ring, and the same
audience-fanout chokepoint. What differs is *how the audience is
resolved* and *where the membership lives*.

## Two channel-document types

The runtime-vs-persistent split shows up at the type level. Chat does
not paper over the lifetime difference with a single union; the two
shapes are distinct classes:

```ts
// lib/social/Channel.ts
export class Channel extends Document {
  static collectionName = 'channels';
  static fieldMeta: FieldMeta = {
    name: { persistent: true },
    kind: { persistent: true },
    subject: { persistent: true },
    procedure: { persistent: true },
  };

  name: string = '';
  kind: ChannelKind = 'player-created';
  subject: string = '';                 // _id of the owning Subject
  procedure: ChannelProcedure = 'open'; // 'open' (cycle 1) | 'ordered'
}

export type ChannelKind = 'player-created' | 'open-join-standalone';
export type ChannelProcedure = 'open' | 'ordered';
```

> **Since forums cycle-1:** `Channel` no longer carries its own
> `owner` / `groupRef`. Identity (the mutation gate) and audience (the
> membership binding) moved onto a shared **`Subject`** record (see
> [forums.md](./forums.md)); a chat channel is now just *one surface*
> on a Subject, addressed via `Channel.subject` (the Subject's `_id`).
> The Channel additionally carries a `procedure` — `'open'` (cycle 1)
> or the deferred `'ordered'` (recognized-speaker discipline; the
> flag ships, the behavior doesn't). ⭐ It is the SAME word the forum's
> organizer uses, because it is the same question: *does a procedure
> govern what may be said and when?* See
> [forums.md § The four surfaces](./forums.md). `Subject.owner` and
> `Subject.groupRef` are the fields that `Channel.owner` /
> `Channel.groupRef` lifted into. See [§ History](#history).

```ts
// lib/social/AdHocChannel.ts
export class AdHocChannel {
  readonly handle: string;
  readonly members: Set<Stuff>;        // live refs, runtime-only
  readonly createdBy: string;
  readonly createdAt: number;
}
```

`Channel extends Document` rides the plain-record persistence track
(see [persistence.md](./persistence.md)); each `findById` rehydrates
a fresh instance and `save()` writes back. `AdHocChannel` is a bare
class — no Document base, no Stuff lifecycle, no persistence — held
only by the in-memory `byHandle` map on the catalogue. Making the
distinction visible at the type level means a function signature can
declare "I take a persistent channel" or "I take an ad-hoc handle"
without a runtime flag check.

## Chat consumes the group substrate

This is the load-bearing architectural decision in the v1 chat
substrate. Player-created channels do not store membership inline;
instead, the channel's `Subject` carries a `groupRef: GroupRef`
(since forums cycle-1; formerly inline on `Channel`) pointing at the
channel's membership source, and audience reads route through
`GroupApi.membersOf(subject.getGroupRef())`. Chat is a **consumer**
of the [grouping](./grouping.md) facade, not a provider — the same
indirection that lets a future channel be backed by an MQL query
("everyone in this guild"), a contacts list ("my friends"), or any
other `GroupProvider` source without the chat layer caring.

For the v1 wave, `chat make <name>` mints a fresh **managed Group**
at create time, stamps the owning `Subject`'s `groupRef` with
`'managed:<groupId>'`, and saves the documents. `chat disband`
cascade-deletes the backing Group. Standalones leave the Subject's
`groupRef` empty — they have no membership concept at all; audience is
computed from per-player subscriptions, not a group.

The chat substrate **owns** the backing Groups it mints. The Group
model knows nothing about chat — that's the right ownership
direction, and the user-facing `group list` view filters out
chat-backed Groups by consulting `ChatApi.getBackingGroupIds()`
(maintained by the catalogue alongside its name cache). A player
who runs `chat make raid-night` does not see a phantom
`raid-night` row in `group list`; the substrate is invisible
unless they ask through the chat verbs.

The earlier round-3 design (membership stored inline on Channel +
chat as its own `GroupProvider`) inverted this — it made chat a
*provider*, which forced cross-cutting consumers to ask the chat
substrate for membership instead of asking the grouping substrate.
That foreclosed the "channel backed by an MQL group / a contacts
list / a guild roster" composition entirely. v1 ships the correct
direction.

## Membership ≠ subscription

The other load-bearing chat-slate concept survives intact. *Membership*
is eligibility — am I in the channel's audience at all? *Subscription*
is reception tuning — am I currently listening, and am I muted?

The two states have different storage and different consumers:

| Concern         | Where it lives                                  | Who reads it             |
|-----------------|-------------------------------------------------|--------------------------|
| Membership      | Backing Group via the `Subject`'s `groupRef` (player-created); n/a (standalone); `AdHocChannel.members` (ad-hoc) | `audienceFor` / `visibleChannels` via `GroupApi`, the post path |
| Subscription    | `SubjectSubscriberMixin` on Avatar, **per-subject** (not per-channel) | `audienceFor`, `chat join` / `leave` / `mute` / `unmute` |

The chat-facing subscription record is small:

```ts
export interface ChannelSubscription {
  tunedIn: boolean;
  muted: boolean;
}
```

Since forums cycle-1, subscription state moved off a per-channel
`PropertiedMixin` key (`chat.subscription.<channelId>`) onto a
**per-subject** store on `SubjectSubscriberMixin` (keyed
`subjectId → { followed, mutedSurfaces }`). `ChannelCatalogue`'s
`getSubscription` / `setSubscription` map the chat-facing
`{ tunedIn, muted }` shape onto the subject's `{ followed,
mutedSurfaces }` (`muted` ⇔ the `'open-chat'` surface being in
`mutedSurfaces`), and migrate any legacy `chat.subscription.<channelId>`
property on first read. See [forums.md](./forums.md) and
[§ History](#history).

Different channel kinds use the two states differently. Open-join
standalones have no member roster at all — every player is eligible
— so `audienceFor` walks every online avatar and filters by
`(tunedIn && !muted)`. Player-created channels filter by both: the
roster gates eligibility, and the per-channel subscription further
gates reception. Ad-hoc channels skip subscriptions entirely —
membership in the in-memory `Set` is the only check, and the cohort
is implicitly tuned-in until the channel disbands.

Persistent-channel subscriptions survive logout (they live on the
Avatar's persistent per-subject `SubjectSubscriberMixin` store);
ad-hoc subscriptions die with the channel.

## `ChannelCatalogue` — the chat runtime chokepoint

Every persistent and runtime piece of chat state lives on a single
singleton Stuff at `/obj/ChannelCatalogue`. It holds three maps:

- `byName: Map<string, Channel>` — persistent-channel cache, keyed
  on lowercased name. Warmed at `postRegister` via `Channel.find({})`
  and refreshed write-through on create / rename / disband.
- `byHandle: Map<string, AdHocChannel>` — runtime-only ad-hoc
  registry. An entry lives until disbanded or promoted to a
  persistent Channel.
- `history: Map<channelId, MessageFrame[]>` — per-channel FIFO ring
  capped at 200 frames, shared across persistent + ad-hoc channels.
  Lost on server restart; persistent history retention is its own
  design problem and stays deferred.

Public method surface:

```ts
// Resolution
resolveByName(name: string): Promise<Channel | null>
resolveHandle(handle: string): AdHocChannel | null
resolveHandleForActor(actor: Stuff, handle: string): AdHocChannel | null

// Ad-hoc lifecycle
openAdHoc(creator: Stuff, members: Iterable<Stuff>): AdHocChannel
disbandAdHoc(handle: string): boolean

// Posting + audience
postToChannel(speaker: Stuff, channel: Channel, body: string): Promise<void>
appendToHistory(channelId: string, frame: MessageFrame): void
historyFor(channelId: string): readonly MessageFrame[]
audienceFor(channel: Channel): Promise<Stuff[]>

// Subscription
getSubscription(avatar: Avatar, channel: Channel): ChannelSubscription
setSubscription(avatar: Avatar, channel: Channel,
                next: Partial<ChannelSubscription>): ChannelSubscription

// Promotion + player-channel CRUD
promoteAdHocToManaged(handle: string, newName: string,
                      promoter: Stuff): Promise<Channel>
createPlayerChannel(owner: Stuff, name: string): Promise<Channel>
disbandPlayerChannel(name: string): Promise<boolean>
renamePlayerChannel(oldName: string, newName: string): Promise<Channel>

// Listing + invariants
visibleChannels(actor: Stuff): Promise<{ persistent: Channel[]; adHoc: AdHocChannel[] }>
reservedNames(): ReadonlySet<string>
```

`ChatApi` (in `mud/api/`) is a thin static facade over these methods.
It carries no state; every call resolves the catalogue lazily via
`StuffApi.findByTemplatePath` / `StuffApi.singleton` and delegates.
The pattern matches `SoulApi` and `GroupApi`: a sanctioned cross-
cutting Api whose job is to thread call-security around a singleton
that does the actual work.

## Posting — `postToChannel`

`ChannelCatalogue.postToChannel(speaker, channel, body)` is the single
audience-fanout chokepoint for persistent channels. The shape:

1. Resolve audience via `audienceFor(channel)` — for `open-join-
   standalone` (or any open Subject), walk every online avatar and keep
   `(tunedIn && !muted)`; for `player-created`, read membership via
   `GroupApi.membersOf(subject.getGroupRef())` (a `'managed:<groupId>'`
   ref today, but any `GroupProvider` source tomorrow) and apply the
   same tunedIn/!muted filter.
2. Compose the speaker's self frame through `MessageApi.scene` so it
   picks up the standard `commandId` / `causingCommandId` attribution
   and the per-channel `meta.channelId` stamp.
3. Fan out to the audience. Scene's `toTarget` is single-recipient;
   chat needs N, so the post path emits per-recipient frames directly
   via `MessageApi.sendMessage` with `audience:witness` tags and the
   same channel meta. **The manual witness + history frames also stamp
   `meta.commandId`** (read once from the ambient command context) — the
   self frame got it free from Scene, but the hand-built frames omitted
   it until the reactions build, which made chat posts un-correlatable
   client-side. With it, a chat post is a reactable act (scope
   `channel:<subject's groupRef, else channelId>`); see
   [reactions.md](./reactions.md).
4. Append the rendered self-frame to the per-channel history ring.

The reason for the split between Scene (self) and direct sends
(audience) is that `Scene` is built around the actor/target/peers
audience taxonomy; chat is a different topology (N named
subscribers, none of whom are room-peers of the speaker). Going
through `MessageApi.sendMessage` directly keeps chat on the same
lone delivery chokepoint without abusing Scene's audience shape.

Ad-hoc posts are handled inline in `ChatController.executePost` —
they iterate the `AdHocChannel.members` set and emit one
`MessageApi.scene(speaker).toTarget(...)` per recipient (the cohort
is small enough that the Scene API fits cleanly), then append to
history. Same topic, same modality, same meta shape — the wire is
indistinguishable from a persistent channel post except for the
channelId being the short ad-hoc handle.

Every chat frame carries:

- `topic: 'speech.channel'`
- `meta.modality: 'verbal-esp'`
- `meta.channelId: string` (channel `_id` for persistent, handle for
  ad-hoc)
- payload `{ channelId, channelName, speaker: StuffRef, text }`

The cockpit groups and routes by `meta.channelId`. Topic strings
stay channel-agnostic — the topic is the *kind* of event ("a chat
post"); the channel identity is metadata.

## Verb shape and the fallthrough flag

`chat` is one verb with two grammars. Management subcommands
(`chat list`, `chat join Help`, etc.) use the standard subcommand
shape; bare posts (`chat gossip hello`) use a flat `<channel>
<message>` shape on the same verb. The two share a single YAML
view by opting into the new **subcommand fallthrough** flag — a
small command-framework extension shipped alongside chat:

```yaml
# mud/cmd/social/chat.yaml
verbs: [chat]
controller: ChatController
validators:
  - /lib/command/validators/requiresAnimate
  - /lib/command/validators/requiresVerbalESP
fallthrough: true
subcommands:
  list:   { description: "List channels visible to you" }
  join:   { args: [{ name: name, type: string, required: true }] }
  leave:  { args: [{ name: name, type: string, required: true }] }
  mute:   { args: [{ name: name, type: string, required: true }] }
  unmute: { args: [{ name: name, type: string, required: true }] }
  who:    { args: [{ name: name, type: string, required: true }] }
  make:   { args: [{ name: name, type: string, required: true }] }
  rename: { args: [{ name: old_name }, { name: new_name }] }
  disband:{ args: [{ name: name, type: string, required: true }] }
  history:{ args: [{ name: name, type: string, required: true }] }
  promote:{ args: [{ name: handle }, { name: new_name }] }
args:
  - name: channel
    type: string
    required: true
  - name: message
    type: string
    required: true
    greedy: true
```

The matcher's behavior — described in
[command-routing.md](./command-routing.md) under Phase 3a — is
two-stage: try to bind the next token as a subcommand from the
`subcommands` map (Phase 2); on `unknown-subcommand`, and only when
the verb declares `fallthrough: true`, fall through to bind tokens
against the top-level `args:` as a flat verb (Phase 3a). The flag
is opt-in; most subcommanded verbs (`measure`, `prompt`,
`settings`, `var`) want the `unknown-subcommand` error as their
failure mode and stay on the original behavior.

`ChatController.execute` decides which grammar matched by reading
`model.subcommand`: when it's absent the controller treats the
input as a bare post (`channel` + `message` populated); when it's
present the controller dispatches per-subcommand.

The `chat make` subcommand cross-checks the requested channel name
against the same reserved set the matcher uses, refusing names that
would collide with a subcommand keyword:

```ts
// ChannelCatalogue.RESERVED_NAMES
new Set([
  'list', 'join', 'leave', 'mute', 'unmute', 'who',
  'make', 'rename', 'disband', 'history', 'promote',
]);
```

The reserved-name check is shared between `createPlayerChannel`
and `renamePlayerChannel`, so the matcher's subcommand-precedence
guarantee is honored on the *creation* side too: a player cannot
mint a channel that the parser would never route bare-post traffic
to.

## Promote — ad-hoc to player-created

The conversational arc chat ships in v1 is: a player `dm`s a few
friends; the multi-target send mints an ad-hoc channel under the
hood; later, when the group decides to stick together, any member
runs `chat promote <handle> <name>` to graduate it. The flow:

1. Look up the ad-hoc via `byHandle.get(handle)`; refuse if the
   target name is already taken.
2. Build the new `Channel` with `kind: 'player-created'`. Each
   ad-hoc member becomes a member of the new channel; the
   promoter becomes `owner`, everyone else lands as `member`.
3. `c.save()` writes the new Document; `byName` updates so the
   channel is immediately resolvable.
4. The runtime history ring keyed under the ad-hoc handle is
   moved (`Map.set` + `Map.delete`) to the new channel's `_id`,
   so the conversation's recent backlog survives the
   graduation.
5. The ad-hoc registry entry is disbanded
   (`byHandle.delete(handle)`).
6. The catalogue's own `postToChannel` fires a system frame
   announcing the promotion. Members receive it on the new
   channel id — their first sign that the conversation has a
   name now.

The promotion is one-way (no demote, no undo) by design — the
conversation gets a name and grows up. Subsequent `dm <oldHandle>`
fails with a clear error (the handle is gone from the registry).

## Bootstrap and seeding

The catalogue self-warms at registration time. Its `postRegister`
hook awaits `Channel.find({})` and populates `byName`, so by the
time any verb resolves a channel by name the cache is hot.

Seed standalone channels — `Help`, `Global`, `Chat` — come from
`packages/server/src/mud/config/channels.yaml`, *not* from
`mud/seeds/`:

```yaml
channels:
  - name: Help
    kind: open-join-standalone
  - name: Global
    kind: open-join-standalone
  - name: Chat
    kind: open-join-standalone
```

`ChannelSeeder.run()` is invoked from `main()` after Mongo connect,
parallel to `EmoteSeeder.run()`. It's insert-only and idempotent:
existing rows are left alone, so an author edit via `chat rename`
survives subsequent boots. The reason channels.yaml does NOT live
under `seeds/` is the same as for emotes — Channel records are
`Document` rows, not Stuff templates, and the `SeederManager` that
walks `seeds/` writes only into the `domain` (template) collection;
putting channel YAML under `seeds/` would mis-route into the wrong
collection.

After Wave 1, the standalone roster is runtime-mutable through the
same wizard-tier verbs that manage player-created channels (`chat
rename`, `chat disband`, `chat make`). The seed YAML is bootstrap
only.

## Deferred — see the chat slate

Chat ships its v1 core under this subsystem doc. The
chat slate is **retained** for everything
in the design space that v1 deliberately defers:

- **Channel role overlay** (`post / invite / manage / moderate /
  admin`) — v1 carries a coarse `owner / admin / member` role on
  the backing Group, but no role-gated actions wire to it yet.
- **The channel config block** (`editPolicy`, `retention`,
  `postPermission`, default notification level) — sensible
  hardcoded defaults in v1; per-channel tuning waits.
- **Edit / delete with "(edited)" markers** — out.
- **Mentions** (`@name`, `@here`, `@channel`) and the offline
  mention inbox — out entirely; no `@` parsing this build.
- **Pinned messages / announcement mode** — out.
- **Notification fine-tuning** beyond mute / unmute — no
  `all / ambient / mentions-only` axis yet.
- **Group-projected channels** — guild chat, party chat, zone
  chat. The grouping substrate exists, but the source systems
  (guilds, parties, zones-as-social) do not yet, so there is
  nothing to project from.
- **Directory / search** beyond `chat list`.
- **Cross-posting / channel bridging.**
- **Anonymity / pseudonymity** on channels.
- **Player-channel succession** beyond "owner leaves → auto-disband
  if no transfer." No mod-promotion, no abandonment GC timer.
- **`chat add <handle> <target>`** for ad-hoc cohort growth —
  promotion is the v1 path; the explicit-add vector is the
  consent-bypass shape sidestepped for now.
- **`chat name <name>`** to rename an ad-hoc channel without
  promoting it — out, to dodge naming politics.

All of these live in the slate's open design space and graduate
into this doc as they ship.

## History

- **Subject retrofit (forums cycle-1, 2026-06).** Chat was retrofitted
  onto the shared **`Subject`** layer the forums build introduced. The
  `Channel` document shed its own `owner` / `groupRef` fields; identity
  (the mutation gate) and audience (the membership binding) now live on
  `Subject` (`Subject.owner` + `Subject.groupRef`), and a chat channel is
  one *surface* on a Subject (`Channel.subject` → `Subject._id`).
  `Channel` gained a `procedure` field (`'open'` cycle-1, `'ordered'`
  deferred — renamed from `'free'` / `'rules-of-order'` in Wave 6, when
  the forum organizer and the chat procedure were unified onto one
  ordered/open axis). Subscription state moved from a per-channel
  `PropertiedMixin` key (`chat.subscription.<channelId>`) to a
  **per-subject** store on `SubjectSubscriberMixin` (`{ followed,
  mutedSurfaces }`); `ChannelCatalogue.getSubscription` /
  `setSubscription` map the chat-facing `{ tunedIn, muted }` shape onto
  it and migrate any legacy key on first read. See
  [forums.md](./forums.md).


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

- [messaging.md](./messaging.md) — the Scene composer, the lone
  `MessageApi.sendMessage` delivery chokepoint, modality stamping,
  and `SensorMixin.filterMessage` (the reception gate that drops
  chat frames for implant-less recipients).
- [forums.md](./forums.md) — the `Subject` layer chat was retrofitted
  onto: identity + audience (`Subject.owner` / `Subject.groupRef`) and
  the per-subject `SubjectSubscriberMixin` subscription store a chat
  channel is now one surface on.
- [grouping.md](./grouping.md) — the `GroupApi` facade that chat
  consumes for channel-membership reads via the Subject's `groupRef`.
- [topics.md](./topics.md) — the `TopicCatalogue` source of truth
  for player-facing topic descriptors; `speech.channel` is
  authored under `seeds/lib/messaging/Topic/`.
- [command-routing.md](./command-routing.md) — Phase 3a and the
  `fallthrough: true` verb-level flag the chat verb relies on.
- [command-spec.md](./command-spec.md) — the author guide for the
  combined `subcommands:` + `args:` + `fallthrough: true` YAML
  shape, with `chat.yaml` as the worked example.
- [persistence.md](./persistence.md) — the `Document` track that
  `Channel` rides on; the `channels` collection alongside `emotes`
  and `groups`.
- [augmentation.md](./augmentation.md) — the baseline AetherImplant
  that makes the `'verbal-esp'` modality universal for players, so
  every chat channel is in-fiction "a frequency on your implant."

## Audience resolution across the sandbox boundary

`ChannelCatalogue.audienceFor` returns the **registry (field) avatar**,
always — even for a subscriber currently inside a circle. Comms are
seamless across the boundary, but the redirect belongs at the delivery
seam (`Avatar.handleMessage` forwards a parked body's frames to
whatever sockets its live vessel holds), not here: a recipient is also
the *viewer* every per-recipient MML name is lensed for, so putting the
vessel in the audience makes a field-context render read circle-
resident perception state, and one person entering a circle killed the
channel post for the whole channel.

The fan-out excludes the speaker **by identity path**, not by object.
Once the audience is field avatars and the speaker may be a vessel,
`a === speaker` stops meaning "the same human being" — a player posting
from inside their own circle received their own line twice, once as
"You" and once as a stranger. Expect this wherever a comparison means
*person* rather than *object*. See [sandbox.md](./sandbox.md).

### Wave 6 (client rebuild) — 2026-08-15

`Channel.procedure` renamed `'free'` → `'open'` and `'rules-of-order'`
→ `'ordered'`, unifying with the forum's organizer onto one axis (see
[forums.md § The four surfaces](./forums.md)). The chat SURFACE in the
client became a terminal that scopes the forum command line via
`cockpit cli --prefix`, rather than carrying an input of its own.
