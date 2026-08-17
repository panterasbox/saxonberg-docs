# Emotes

The emote subsystem turns expressive acts (`wave`, `smile`, `bow`,
`cry`, free-form `:scratches his chin`) into the same per-audience
`Scene` plumbing that powers `say` and `tell`, with one significant
difference: emotes ride the ESP modality (`emotive-esp`) and an
authored catalog of grammar templates rather than a single fixed
prose form. The substrate ships as part of the social-cluster build
(2026-06): a `SoulMixin` on every `Character`, an `Emote` Document
catalog backed by MongoDB, a `SoulCatalogue` singleton holding the
runtime verb→Emote cache, and a thin `SoulApi` facade consulted by
the verb dispatcher. The slate at
emotes-slate.md stays live as the
register of design space not yet built.

## What emotes are vs. what they aren't

An emote is an **expressive act**: a character performs a gesture or
display, and the act is perceived as a single coherent event by
everyone in the audience. The wire body is server-rendered natural
language (`Bobalu waves at Iffy.`), composed once on the server with
the actor's identity attributed structurally via `<name>`. Emoji,
when present, ride as a payload glyph (`👋`) for the client to
render alongside or in place of the prose — never as the primary
representation.

Emotes are NOT speech. Speech rides `VocalMixin` (acoustic, propagates
through the `SoundModality` walk, gated by atmospheric medium and
hearing organs); emotes ride the **ESP carrier** stamped as
`meta.modality = 'emotive-esp'` and gated at
`SensorMixin.filterMessage` by whether the recipient's sensorium
includes that modality.

`SoulMixin` is **composed natively** on every `Character` (Avatar +
NPC) and is not augment-gated. The diegetic story is that *making*
expression is innate to a sentient being; only the ESP carrier that
conveys it remotely is augment-gated, and that gating lives on
`AetherMixin` (which `Avatar` composes by char-gen convention; NPCs
opt in per-class). Local emote works without any implant because
the in-room peer audience is decided by Containable scope, not by
ESP propagation.

NOT for: speech (use `VocalMixin`); aether comms (use `AetherMixin`);
the per-recipient sensorium check (the modality stamp + the shipped
`SensorMixin.filterMessage` do that automatically).

## The `Emote` Document

Authored catalog records live at `lib/social/Emote.ts`, a plain
`Document` subclass mapped to the `emotes` MongoDB collection (added
to the `Collections` enum). The shape is deliberately flat:

```typescript
export class Emote extends Document {
  static collectionName = 'emotes';
  static fieldMeta: FieldMeta = {
    verb: { persistent: true },
    aliases: { persistent: true },
    grammar: { persistent: true },
    echo: { persistent: true },
    emoji: { persistent: true },
    tags: { persistent: true },
    valence: { persistent: true },
  };

  verb: string = '';
  aliases: string[] = [];
  grammar: EmoteGrammar = { slots: {}, template: '' };
  echo: EmoteEcho = 'default';
  emoji?: string;
  tags: string[] = [];
  valence: number = 0;
}
```

`verb` is the canonical lookup key (unique-indexed). `aliases` is a
narrow exception for true synonyms (`greet` ↔ `hi`, `hello`); the
convention is to leave it empty. `grammar` carries the Liquid
template plus per-slot metadata (see next section). `echo` reserves
the future remote-emote echo policy. `emoji` is the optional single
glyph, surfaced on the payload alongside the failsafe prose body.
`tags` is the reactions / classification hook — now consumed: the
reaction layer groups chips by `tags[0]` (see
[reactions.md](./reactions.md)). `valence` is the **signed renown value**
of the emote *as a reaction* (esteem `+` / notoriety `−`; default `0` =
neutral) — the renown recompute reads the verb→valence map via
`SoulApi.all()`. Per-emote valence lives here, on the emote, deliberately
*not* as a central config map (seeded in `emotes.yaml`: cheer/applaud/
clap/laugh/grin/agree `+1`, scowl/glare/frown/disagree/groan `−1`). See
[renown.md](./renown.md).

`Emote` is a **catalog record, not a Stuff**. Plain data records
that don't participate in the clone pipeline live on the Document
track — cheap reads, no construction sentinel, no proxy. Catalog
records don't ride the clone-pipeline HMR path; author edits via
`soul edit` are write-through to both Mongo *and* the live
`SoulCatalogue` cache, so same-process authoring lands immediately.
Cross-process cache invalidation is a known v1 limitation.

## The grammar substrate

`EmoteGrammar.ts` defines the authoring vocabulary and the runtime
binder + renderer. Two slot kinds: `stuff` slots resolve via MQL to
a `Stuff` ref (typically the emote target), `free` slots accept
arbitrary text (manner clauses, adjectives, fragments).

```typescript
export type SlotKind = 'stuff' | 'free';

export interface EmoteSlot {
  kind: SlotKind;
  optional?: boolean;
  scope?: string;  // for `stuff` slots; default 'online'
}

export interface EmoteGrammar {
  slots: Record<string, EmoteSlot>;
  template: string;
  verbForm?: { self: string; third: string };
}
```

A single Liquid template per emote handles every slot-presence
permutation via `{% if %}` conditionals — no four-permutation
matrix, no template-key cascade. The renderer pre-binds
audience-specific variables before invoking `ProseApi.format`:
`actor` swaps "You" / `<name>`, `target` swaps "you" / `<name>`,
and the verb-conjugation helpers `s` / `es` / `ies` swap `""` /
`"s"` and `"y"` / `"ies"`. A wave grammar looks like this:

```yaml
- verb: wave
  emoji: "👋"
  tags: [greeting]
  grammar:
    slots:
      target: { kind: stuff, optional: true }
    template: "{{ actor }} wave{{ s }}{% if target %} at {{ target }}{% endif %}."
```

Rendered for the self audience with no target: `You wave.` For the
peer audience with `target = Iffy`: `Bobalu waves at Iffy.` For
Iffy's target audience: `Bobalu waves at you.` Irregular verbs
declare the optional `verbForm` override which exposes a `verbForm`
variable to the template.

`EmoteGrammarRunner.bind` walks the speaker's positional tokens
against the slot map in YAML order, consuming one token per slot.
`stuff` slots resolve through `MqlApi.resolveOne` at the declared
scope (default `'online'`). Optional `stuff` slots skip-on-mismatch,
so `wave happily` binds against the same grammar as `wave iffy`:
the optional `target` slot fails to resolve "happily" as a Stuff,
the cursor moves on, and a following `free` slot consumes the
remainder. The last slot in YAML order is greedy if it's `free`.

`EmoteGrammarRunner.render` builds the per-audience context and
runs the template through `ProseApi.format`. The Mml-aware escape
ensures Mml fragments emit verbatim while raw strings (slot fills)
get the five-entity escape.

## `SoulMixin` — rendering and verb-side send

`SoulMixin` composes onto `Character` and exposes two render
methods plus two send methods:

```typescript
export interface Soul {
  renderEmote(emote: Emote, opts?: EmoteOptions): EmoteBodies;
  renderFreeForm(text: string, target?: Stuff): EmoteBodies;
  emote(emote: Emote, opts?: EmoteOptions): void;
  emoteFree(text: string, target?: Stuff, inReactionTo?: string): void;
}
```

`EmoteOptions` carries an optional `inReactionTo?: string` (and
`emoteFree` a matching 3rd arg): the `commandId` of a prior act this
emote *reacts to*. When set, the send pokes the reaction layer
(tally/threshold/suppression) before composing the Scene; absent, it's
an ordinary emote. A non-reaction emote is itself a reactable act. The
reaction substrate rides this one send path wholesale — see
[reactions.md](./reactions.md).

`EmoteBodies` is the per-audience triple `{ self: Mml, peer: Mml,
target?: Mml }`. `renderEmote` runs the grammar renderer three
times (self / peer / target) and returns the triple; `renderFreeForm`
wraps the player-supplied text in `Mml.markdownToMml` with the
speaker's perceiver mention resolver, then composes the self / peer
bodies inline (`You ${parsed}.` / `${actorName} ${parsed}.`). The
target frame, when there is a target, reuses the peer body because
the cockpit's per-viewer rendering substitutes "you" for the
target's own `<name>` / `<mention>` match.

The `emote` / `emoteFree` methods are the in-room verb-side default,
parallel to `VocalMixin.say`. The peer-audience routing follows the
**Containable-wins rule**: an actor that's Containable broadcasts
to peers; a pure-Container actor (a haunted location) falls through
to `toContents`.

```typescript
emote(emote: Emote, opts?: EmoteOptions): void {
  const actor = this as unknown as Stuff;
  const bodies = this.renderEmote(emote, opts);
  const scene = MessageApi.scene(actor)
    .topic('act.emote')
    .modality('emotive-esp')
    .toSelf(bodies.self)
    .payload({ verb: emote.verb, emoji: emote.emoji, tags: emote.tags });
  if (opts?.target && bodies.target) {
    scene.toTarget(opts.target, bodies.target);
  }
  if (MixinApi.isContainable(actor))      scene.toPeers(bodies.peer);
  else if (MixinApi.isContainer(actor))   scene.toContents(bodies.peer);
  else throw new Error('...');
  scene.send();
}
```

The rendering surface (`renderEmote` / `renderFreeForm`) lives on
the mixin **regardless of routing**. Channel-routed and
DM-handle-routed emote paths call the render methods to get the
triple, then compose their own `Scene` with a different audience
(channel members instead of in-room peers). The mixin is not a
router; rendering consistency lives in the mixin, audience
computation lives at the dispatch site.

`SoulMixin` also contributes the `social.emote.render` setting
(`text | emoji | both`, default `both`) to the `EnvironmentMixin`
schema. The server always emits both shapes; the client picks per
setting.

`SoulMixin` further owns `introduceSelf()` — the shared, modality-neutral
introduce act (the introduce scene + `learnIdentity` to all in range)
that the `introduce` verb, the `introduces` NPC brain, and the player
auto-introduce hook all route through — plus the `social.autoIntroduce`
setting (default off) that opts a player into introducing on arrival. See
[npc-dialogue.md](./npc-dialogue.md) and [belief.md](./belief.md).

## `SoulCatalogue` + `SoulApi`

The verb→Emote runtime cache lives on a Stuff singleton,
`/obj/SoulCatalogue`, sibling to `/obj/TopicCatalogue` and
`/obj/EventRegistry` per the singleton-in-`obj/` convention. The
catalogue extends `Idea` composed with `PostRegistrationMixin` —
the `postRegister` hook warms `cache: Map<string, Emote>` from
Mongo via `Emote.find({})` after `EmoteSeeder.run` has populated the
collection. The map is keyed by canonical verb AND by every alias.

The catalogue carries the live mutation surface (`resolve` / `mint`
/ `edit` / `delete` / `all`). `mint` writes a new `Emote` via the
Document track *and* indexes it into the cache; `edit` mutates in
place, re-runs `save()`, and re-indexes; `delete` drops cache
entries and `Document.delete()`s the record. `canDestruct` refuses —
`Application` resolves the catalogue lazily and expects it to stay
live for the process lifetime.

```typescript
public async resolve(verb: string): Promise<Emote | null> {
  const map = await this.ensureCache();
  return map.get(verb.toLowerCase()) ?? null;
}
```

`SoulApi` (`mud/api/soul.ts`) is the thin caller-facing facade. Its
five methods (`resolve` / `mint` / `edit` / `delete` / `all`) are
1:1 delegates to the catalogue singleton, plus `invalidateCache`
for the wizard `reload` flow. The Api holds **no state of its own** —
the catalogue Stuff is the live state, and `SoulApi.#catalogueRef`
is just a cached pointer resolved through
`StuffApi.findByTemplatePath`. The class ends with
`SecurityApi.decorateApiClass(SoulApi)`. The shape mirrors
`TopicCatalogue` exactly: a Stuff singleton owns the data, an Api
thin-wraps it for the stable caller-facing surface.

## Dispatch paths

Three distinct dispatch paths, each entering the rendering surface
at a different door:

**(a) Bare-verb router catalog (`wave iffy`).** Unknown verbs are
the catalog's main entry point. In `CommandGiver._runChain`, when
the affordance verb-match list (`getAffordances()` filtered by verb)
comes back empty, the router checks `MixinApi.isSoul(speaker)` and consults
`SoulApi.resolve(parsed.verb)` before emitting `unknown-verb`. On
a hit, the router binds slots via `EmoteGrammarRunner.bind`, calls
`speaker.emote(emote, { target, fills })` inline, and returns —
**no controller, no synthesized command model, no YAML view**.

**(b) Emote-prefix dispatch (`:wave iffy` / `;wave iffy`).** The
`msh` parser (`lib/command/parsers/msh.ts`) has a single
pre-tokenization hook: `detectEmotePrefix` matches a leading `:`
or `;` followed by a verb-starting char (letter or underscore).
ASCII smileys like `chat foo :)` are *not* mis-dispatched because
the second char must begin a verb word. When the prefix matches,
the parser strips the sigil and stamps `parsed.emotePrefixed =
true`. The router branch in (a) then tries the catalog first; on a
**miss** with the prefix flag set, the router falls back to
`speaker.emoteFree(parsed.source.trim())` — `;waves at iffy.`
becomes a free-form emote when "waves" isn't a catalog verb.

**(c) Free-form `emote <text>` verb.** A normal YAML verb
(`emote.yaml`) dispatches to `EmoteController`, which checks for
`SoulMixin` composition and calls `speaker.emoteFree(model.body)`.
The controller exists *only* for this case; the bare and prefixed
paths run inline in the router.

## The `soul` authoring suite

`SoulController` (`obj/command/social/SoulController.ts`) implements the
wizard-tier author surface: `soul make / edit / delete / show /
list`. The verb is gated by composition — `AuthorMixin` contributes
`social/soul.yaml` to its `commandContributions.self`, paralleling the
existing `clone` / `reload` / `eval` / `teleport` AuthorMixin
verbs. When the access slate lands the gate moves mechanically to
`AccessApi.can(actor, 'soul.author', null)`.

`soul make <verb> <yaml-or-json-spec>` mints a new `Emote` via
`SoulApi.mint`. The spec is a greedy remainder; both YAML and JSON
parse paths are tried. `soul edit <verb> <field> <newvalue>`
updates one of `template` / `grammar` / `aliases` / `tags` /
`emoji` / `echo` and re-saves through `SoulApi.edit` (which both
persists and re-indexes the cache). `soul delete` drops from cache
and Mongo; `soul show` prints the record; `soul list` enumerates
the cache.

Every author edit is write-through to **both** Mongo and the
`SoulCatalogue` cache, so a freshly minted `bogleg` is immediately
resolvable on every dispatch path without restart.

## Topic and modality

Emote frames stamp `topic = 'act.emote'`, a single
leaf for both catalog and free-form emotes; payload distinguishes
(`{ verb, emoji, tags }` for catalog, `{ freeForm: true, text }` for
free-form). The leaf is authored as a `Topic` template at
`seeds/lib/messaging/Topic/act.emote.yaml`:

```yaml
class: /lib/messaging/Topic
hydratorClass: /obj/persistence/PersistentHydrator
data:
  topic: act.emote
  family: act.emote
  label: Emote
  description: A character performs an expressive act conveyed via the ESP modality.
```

`TopicCatalogue` self-loads it at boot; the cockpit's filter
drawer / gutter tooltip / topic-aware styling all consume the
descriptor with no further wiring.

The modality stamp `meta.modality = 'emotive-esp'` rides every
emote frame via `.modality('emotive-esp')` on the Scene. Reception
gating is the shipped per-frame modality system at
`SensorMixin.filterMessage`: recipients whose
`PerceptionApi.sensorium` doesn't include `emotive-esp` drop the
frame, with the actor's self-frame bypassing unconditionally.
Players receive emotes universally because the baseline
`AetherImplant` confers
`_grantsModalities = ['verbal-esp', 'emotive-esp']` during the
clone cascade; non-implant NPCs miss emotes the same way they miss
DMs. See [augmentation.md](./augmentation.md).

## Universal ESP target delivery

A catalog emote's `stuff` slot resolves at `scope: 'online'` by
default — the target may be in any room, on any zone, anywhere a
spawned `Avatar` or NPC exists. When the slot binds, the Scene's
`.toTarget(target, bodies.target)` call routes the target frame
through `MessageApi.sendMessage` directly to that recipient
regardless of room boundaries. The peer audience (`.toPeers`) still
uses Containable scope; only the targeted frame crosses rooms.

This is the slate's "smile at iffy across the world just works"
model. ESP-channel communication between two beings who share the
comm implant doesn't need line-of-sight or room adjacency; the
recipient's sensorium gates reception independently of where they
happen to be standing. The same `scope: 'online'` shape powers
`dm`'s target resolution.

## Starter roster + bootstrap

The starter catalog (~35 entries) lives at
`packages/server/src/mud/config/emotes.yaml`. The location matters:
the file lives under `mud/config/`, **not** under `mud/seeds/`. The
generic `SeederManager` walks `seeds/*` and inserts every YAML it
finds as a `Template` into the `domain` collection. Emote records
aren't Stuff templates and don't belong there; keeping the YAML out
of the seeds tree avoids double-insertion.

`EmoteSeeder.run()` reads the YAML, walks the entries, and inserts
each into the `emotes` collection if no record with that `verb`
exists. **Insert-only by design** — re-runs are idempotent; author
edits via `soul edit` survive subsequent boots. Dev workflow when
an entry changes: `db.emotes.deleteOne({verb: '<name>'}); restart`,
matching the existing `SeederManager` pattern.

The seeder runs from `main()` after `PersistenceManager.connect`
(which creates the `verb` unique index and the `aliases` index) and
before `BootstrapManager.run` (which warms the `SoulCatalogue`
singleton — the catalogue reads the just-populated collection at
its `postRegister` hook). The starter roster covers greetings, joy /
approval, displeasure, surprise, playful, and abstraction registers.

## The client read face — `SoulApi.snapshot()` (Wave 6)

The catalogue had one reader surface, `soul list`, gated
`requiresCoreAccess` — the **authoring** face. Players need to see the
palette they can already type, so Wave 6 added a player-readable
projection.

`SoulCatalogue.snapshot()` → `EmoteCatalogueEntry[]`: canonical verbs
only (the warm cache indexes every alias to the same `Emote`, so
iterating it directly would emit one grid cell per alias — the aliases
ride their canonical entry), each with its emoji, tags and declared slots
in **declaration order**, which is the order `EmoteGrammarRunner.bind`
consumes tokens in. `required` is derived from the author's `optional`
flag rather than restated. `soul list` keeps its gate: seeing the palette
is not authoring it.

### ⭐⭐ It is fetched, not pushed — `GET /api/emotes`

It rode `ConnectionEstablishedPayload` first, beside `topicCatalogue`.
**That is the wrong home for something expected to grow**: `Avatar.enter()`
runs on **reconnect** as well as fresh login, so a flaky connection
re-shipped the whole catalogue every drop — onto a packet already
carrying the topic catalogue, the news window and the frame backfill.

The catalogue is authored, global, and changes only when somebody runs
`soul make`, which is the textbook case for HTTP caching. So
`backend/EmoteRoutes.ts` serves it read-only on the {@link HelpRoutes}
pattern with a **strong ETag over the payload**, and the client fetches
**lazily, on the first reaction affordance opened**:

| | bytes |
|---|---|
| reconnect | 0 |
| later session, unchanged catalogue | 0 (`304`) |
| player who never reacts | 0 |

⚠ The ETag is a hash of the payload, not a version counter: `soul mint` /
`edit` / `delete` write through to the cache, and a counter would be a
second thing to remember to bump. A hash cannot drift from what it
describes.

⚠ `Cache-Control: private, no-cache` means *revalidate*, not *do not
store*. A `max-age` would serve a stale catalogue for its duration, and
an author who mints an emote should see it next session rather than next
hour.

⚠ **`reactableTopics` deliberately stays on the connection payload.** It
is four strings and it gates whether the `＋` exists at all; a client
that had to fetch before deciding whether to offer anything would flash
the affordance in and out.

⚠⚠ **A failed fetch is not an empty catalogue.** The route answers `503`
rather than `[]`, and the client holds a distinct `failed` state. An
empty catalogue is a legitimate answer — a world with nothing authored —
which the picker renders as "no emotes authored yet"; a failure that
looked identical would be an unwired state wearing an empty state's
clothes, and the client would cache the lie.

⭐ The client draws its picker from this **and from nothing else**. It
previously drew from a hardcoded six-entry `{ verb, emoji }` array; a
hardcoded pairing drifts from the catalogue with nothing failing when it
does. See [reactions.md § The emote picker](./reactions.md).

## What's deferred

Each of the following stays in emotes-slate.md
for a future build. The data shape ships now where deferring would
later force a migration; the **behavior** doesn't run in v1.

- **Layer 2 client-side emoji rendering.** The
  `social.emote.render` setting and the `emoji` payload key both
  ship. The client-side toggle (text / emoji / both per channel) is
  incomplete; v1 always renders the prose body.
- **Layer 3 honorary / entitlement gating.** The Twitch-flavored
  status-badge model (instructor-only emotes, cohort emotes,
  achievement-gated glyphs) is out entirely. No `requires` field on
  `Emote`, no predicate seam. When the first real entitlement
  source ships, it brings its own schema field; document migration
  for one optional field is cheap.
- **Layer 4 reactions + tags aggregation.** The `tags` field on
  `Emote` reserves the aggregation hook, but no `react` verb,
  message-id surfacing, or client aggregation UI ships.
- **Echo routing.** The `Emote.echo` field and the parallel
  `social.emote.echo` setting reserve the data shape, but the
  *second-audience shadow performance* — bystanders in Bobalu's
  current room seeing him performing a wave even though the wave
  was a channel emote — doesn't run in v1. The canonical
  channel / DM audience ships now; the in-room echo waits.
- **Live cross-process HMR.** Author edits via `soul edit`
  invalidate the local-process cache write-through; multi-process
  authoring needs a restart in v1.
- **Wizard-tier authoring GUI.** v1 ships the `soul` verb suite. A
  rich web editor that layers on the same verb-and-Api substrate is
  anticipated but not in this build.

## Related

- [messaging.md](./messaging.md) — `MessageApi.scene` /
  `MessageFrame` / `SensorMixin` / the lone delivery chokepoint;
  emotes invent no new delivery plumbing.
- [augmentation.md](./augmentation.md) — `AetherMixin`,
  `_grantsModalities`, the baseline `AetherImplant` that confers
  `'emotive-esp'` to every Avatar.
- [senses.md](./senses.md) — per-frame modality attribution at
  `Scene.modality(name)` + `SensorMixin.filterMessage`, sensorium
  composition, the seven `Modality` singletons.
- [command-routing.md](./command-routing.md) — the dynamic-verb
  resolver branch in `CommandGiver._runChain`, the per-giver
  recency stack, schema delivery.
- [topics.md](./topics.md) — `TopicCatalogue`, authored topic
  descriptors, the `act.emote` leaf.
