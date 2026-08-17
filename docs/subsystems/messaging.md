# Messaging

The messaging subsystem turns "X happened in the world" into one or more
typed wire frames delivered to the right Sensors and on out to connected
clients. The pipeline:

```
Game object
    composes via mixin sugar (Layer 4) or direct Scene (Layer 3)
        ↓
Scene.send()
    builds one MessageFrame per audience, stamps commandId / causingCommandId
        ↓
MessageApi.sendMessage(recipient, frame)         ← lone delivery chokepoint
        ↓
Sensor.onMessage(frame)
    ├─ filterMessage(frame)   shadowable extension point
    └─ handleMessage(frame)   subclass override
        ↓
Avatar.handleMessage(frame)
    for each connected Interactive
        ↓
Application.sendMessageToInteractive(interactive, frame)
        ↓
Backend.sendMessageToSocket(socketId, frame)
        ↓
WebSocket.send(JSON.stringify(frame))
```

The Sensor → MessageApi → Application → Backend skeleton is the entire
delivery path. There are no parallel channels, no log sinks, no bus-level
sidecars. Anything wanting to consume a frame is, or shadows, a Sensor.

The response-envelope channel follows the same pipeline shape, with
parallel surfaces at every tier: `MessageApi.sendEnvelope` is the lone
delivery chokepoint, `Sensor.onEnvelope` / `filterEnvelope` /
`handleEnvelope` mirror the message triad, and the fan-out runs through
`Application.sendEnvelopeToInteractive` →
`Backend.sendEnvelopeToSocket`. See
[response-envelope.md](./response-envelope.md).

## Nomenclature

These terms have specific meanings; code, comments, and docs use them
precisely.

- **Message frame** — the unit of outbound communication. A typed
  object delivered to a Sensor. Carries a topic, optional tags, an MML
  body, optional structured payload, and metadata. See "Wire envelope".
- **Topic** — *intrinsic* hierarchical classification of a message's
  nature/content. Mandatory. Framework-defined and stable. Dot-separated
  path. Example: `speech.vocal`. Answers "what kind of message is
  this?"
- **Tag** — orthogonal flat property attached to a frame.
  Namespace-prefixed string (`audience:witness`). Open set.
- **Scope** — producer-side concern, NOT a property of the message
  itself. Describes how a producer chose to deliver the frame (to self,
  to peers, to contents, to a specific target). Once delivered, scope
  is gone — the recipient just has a frame.
- **Sensor** — game object that receives frames. The sole recipient
  type. Existing concept (`SensorMixin`). Avatars and NPCs are sensors;
  custom sensors (debug consoles, audit observers) can be written as
  needed.
- **Body** — MML-formatted string representation of a message. Always
  present on every frame. The console-facing face. Even frames primarily
  consumed by widgets carry a body for accessibility, log capture, and
  fallback display.
- **commandId** — live-command attribution. Present on every frame
  composed during the *synchronous* execution of a command. Travels
  with the frame to every recipient (including witnesses who never
  issued the command).
- **causingCommandId** — causal attribution. Present on every frame
  composed inside work descended from a command — sync execution OR
  async aftermath propagated via opt-in schedulers. In sync execution
  this equals `commandId`; in async aftermath `commandId` is absent
  but `causingCommandId` carries the originating command's id.

Topic and scope are orthogonal and never collapsed into a single field:

| Concept | Where it lives        | Required               |
|---------|-----------------------|------------------------|
| topic   | on the frame          | yes                    |
| scope   | producer call only    | producer-side decision |

## Wire envelope

`MessageFrame<T>` lives in `@saxonberg/types` so server and client
agree on shape:

```typescript
interface MessageFrame<T = unknown> {
  id: string;
  topic: string;
  tags?: string[];
  body: string;          // MML markup
  payload?: T;
  meta: {
    timestamp: number;
    commandId?: string;
    causingCommandId?: string;
    frameId?: number;    // per-Interactive monotonic; absent at compose-time,
                         // stamped at Application.sendMessageToInteractive
  };
}
```

`meta.frameId` is absent at compose-time (`Scene.send` and `MudlogApi`
don't set it) and populated per-Interactive at the
`Application.sendMessageToInteractive` send-time chokepoint. Producers
ignore it; consumers (the client) use it for gap detection across both
channels — see [response-envelope.md](./response-envelope.md) for the
shared ordering primitive.

`StuffRef` is the wire-safe Stuff reference. Direct Stuff objects never
cross the boundary:

```typescript
interface StuffRef {
  stuffId: string;
  displayName?: string;  // pre-resolved server-side at compose time
}
```

`MessageApi.refOf(stuff)` builds one. `displayName` is resolved at
compose time via `Stuff.getPresentation()`, so the wire payload
doesn't depend on a re-resolution step on the client.

Frame `id` is a `nanoid()` minted inline at frame construction in
`Scene.send()` and `MudlogApi.*`. No dedicated `IdApi` wrapper —
single-method utility doesn't pay for itself.

When per-audience payloads diverge, the topic's payload type is a
discriminated union keyed by `audience`; the frame's `audience:` tag
discriminates and consumer code narrows via TS pattern matching.

## Response envelope

Alongside `MessageFrame`, the wire carries a second frame shape: the
**response envelope**, a structured machine-readable signal for
dispatch outcomes (`type: 'dispatch-response'` and reserved
`'activity-update'` / `'prompt'` siblings). Envelopes carry no prose
— their payload is `outcome.status` plus a typed `notes` list — and
travel through a parallel Sensor pipeline (`onEnvelope` /
`filterEnvelope` / `handleEnvelope`) with `MessageApi.sendEnvelope` as
the lone delivery chokepoint. Both channels share the
`Interactive.nextFrameId` counter so the client sees gap-free monotonic
ordering across all server → client traffic on a single Interactive.
Full shape, note kinds, and the auto-escalation table live in
[response-envelope.md](./response-envelope.md).

## Topics

Two roots. Dot-separated paths. Lowercase.

```
world.                          # in-fiction
  speech.
    say
    tell
  perception.
    look
    inventory
    scry
    locate
  narration.
    movement                    # walking-style depart/arrive
    teleport                    # sudden, magical in/out
    action                      # generic in-fiction action prose
  identity.
    change                      # name / appearance changes

system.                         # out-of-fiction infrastructure
  connection.
    established
    lost
  auth.
    success
    failed
  command.
    error                       # dispatcher-emitted framework-failure prose
  shell.
    fs                          # filesystem-style verbs (cd, ls, pwd, cat …)
    author                      # author-tier verbs (clone, reload, eval, teleport)
    help                        # help system
    movement                    # shell-tier movement narration (go-style verbs)
  log.
    <category>.<level>
    # category: hot-reload, security, command, persistence, etc. (open)
    # level:    trace | debug | info | warn | error | fatal
```

Conventions:

- All paths lowercase, dot-separated.
- Leaves should be specific enough that a single MML template applies
  (`speech.vocal` is a leaf; `speech` is not).
- Adding a new topic requires no framework changes — producers just emit
  the new topic string.
- Controller-side failures aren't a special topic. A `look` that
  found nothing still composes prose at `sense.look`;
  failure is captured in the dispatch-response envelope's
  `outcome.status` + `notes`, AND in the prose the controller
  fires at the same domain topic it would have used on success.
- **Framework-side** failures (parse, MQL, validator,
  controller-throw) DO have a special topic — `shell.error`
  — populated by the dispatcher's end-of-execute prose sweep. The
  player sees WHY a bad command was rejected without the client
  needing to render envelopes. See
  [response-envelope.md § Envelope vs scene split](./response-envelope.md#envelope-vs-scene-split).

Topic strings are emitted as **dotted-path string literals** at call
sites:

```typescript
MessageApi.scene(speaker)
  .topic('speech.vocal')
  ...
```

The `MessageApi.Topics` constant tree (the earlier shape) was
**retired** during the message-rendering build — the parallel data
in code + YAML led to two sources of truth. The authored source of
truth for the topic vocabulary lives on per-topic YAML leaf Ideas
under `seeds/lib/messaging/Topic/`. The `TopicCatalogue` singleton
self-loads them at boot and turns each topic string into a
player-facing descriptor (friendly label, description, family for the
cockpit's filter drawer, gutter tooltips, etc.). See
[topics.md](./topics.md).

`'shell.diagnostic'` is the prefix used for "all log frames" matching;
`shell.diagnostic` is the framework-emitted **input-echo** topic
(see "Input echo at `shell.diagnostic.*`" below).

## Tags

Open-set, flat, namespace-prefixed strings. Producers attach as
`tags: string[]` on frames.

`audience` is the only namespace defined in v1. It's load-bearing for
the discriminated-union payload pattern and for audience-aware filtering.
Other namespaces (modality, urgency, context, locale, etc.) are
introduced when first feature actually needs them.

```typescript
MessageApi.Tags.Audience.Actor       // 'audience:actor'
MessageApi.Tags.Audience.Target      // 'audience:target'
MessageApi.Tags.Audience.Witness     // 'audience:witness'
MessageApi.Tags.Audience.Bystander   // 'audience:bystander'
```

The Scene composer auto-attaches the appropriate `audience:` tag per
audience frame. Producers don't tag manually.

## MML — Mud Markup Language

`Mml` (`mud/api/mml.ts`) is the composer for body strings. Layer 2 of
the composer stack.

```typescript
class Mml {
  // Constructor is private. Two factory paths:
  static compose(strings, ...values): Mml      // safe — escapes raw values
  static fromMarkup(raw: string): Mml          // explicit trust assertion

  // Identity vocabulary
  static actor(stuff: Stuff): Mml              // → <player> | <npc> | <thing>
  static thing(stuff: Stuff): Mml              // <thing stuff-id="…">…</thing>
  static player(stuff: Stuff): Mml             // <player stuff-id="…">…</player>
  static npc(stuff: Stuff): Mml                // <npc stuff-id="…">…</npc>
  static speech(text: string | Mml): Mml       // <speech>"…"</speech>
  static location(stuff: Stuff): Mml           // <location stuff-id="…">…</location>
  static direction(d: string): Mml             // <direction>…</direction>
  static exit(exit: Exit): Mml                 // <exit dir="…" stuff-id="…">…</exit>

  // Chat / messaging vocabulary
  static chan(id: string, label: string): Mml  // <chan id="…">[Label]</chan>
  static msg(body: string | Mml): Mml          // <msg>…</msg>
  static mention(stuffId: string, label: string): Mml // <mention stuff-id="…">@…</mention>
  static link(href: string, label: string | Mml): Mml // <link href="…">…</link>

  // Color / fragment primitives
  static color(value: string, body: string | Mml): Mml // <color value="…">…</color>
  static text(value: string): Mml              // escaped-once text fragment

  // Emphasis (Discord-dialect markdown subset)
  static strong(body): Mml; static em(body): Mml
  static code(text): Mml;   static pre(text): Mml
  static blockquote(body): Mml; static strike(body): Mml
  static unorderedList(items: Mml[]): Mml
  static orderedList(items: Mml[]): Mml
  static li(body): Mml

  // Long-form (the article dialect — see wiki.md)
  static heading(level: 1|2|3, body, anchor?): Mml  // <h1..3 anchor="…">
  static table(rows: Array<Array<string|Mml>>, header?): Mml
  static spoiler(level: 0|1|2|3, body): Mml         // the APPETITE half only

  // Tree access (the wiki render pipeline resolves over nodes)
  static parseTree(body: string): readonly MmlNode[]
  static serialize(nodes: readonly MmlNode[]): string
  static isKnownTag(tag: string): boolean       // markup vs component
  static componentCandidate(tag: string): boolean

  // Joining
  static list(items: Mml[], opts?: { style? }): Mml  // commas/and or block

  // Projection
  static escape(text: string): string          // five-entity escape
  static stripTags(body: string): string       // bare-plain (drops emphasis)
  static flatten(body: string): string         // markdown-preserving

  // Markdown pipeline (server-authoritative, see message-rendering.md)
  static markdownToMml(text: string, resolver?: MentionResolver): Mml
  static perceiverMentionResolver(speaker: Stuff): MentionResolver
  static channelMentionResolver(participants: Iterable<Stuff>): MentionResolver

  toString(viewer?): string
  toJSON(): string
}
```

An identity ref (`name`/`item`/`object`/`player`/`npc`/`location`)
renders its label through the target's **`Stuff.getPresentationMml(label)`**
— the `Mml`-fragment sibling of `getPresentation`. `Mml.ref` resolves
the viewer-aware label (recognition stays in the render layer) and
hands it in; the default returns `null`, and `Mml.ref` wraps the label
in `Mml.text` (escaped once). An object can override it to return a
richer fragment — a name composed as markup — e.g. the TPA terminal
wrapping its name in `<color>` to tint by status. So a name is a
composable fragment like everything else; `getPresentation` stays a
plain string for non-prose consumers (logs, notes, MQL scalars).

⚠ **`markdownToMml` has two dialects**, selected by an options bag:
**chat** (the default — every hot-path caller, byte-identical to what it
has always emitted) and **article** (`longForm: true` — headings with
sticky anchors, indent-nested lists, pipe tables). An options bag rather
than a fork, because a forked long-form parser would drift from the chat
one silently. `api/__tests__/mml.corpus.test.ts` pins the chat path
byte-for-byte; a diff there is a failure regardless of whether the new
output reads better. See [wiki.md](./wiki.md).

⚠ **`<spoiler>` is the APPETITE half only.** Content above a reader's
*capability* is deleted server-side and never reaches a tag, so if one
is being emitted the reader is already entitled to what is inside. The
two axes are not the same axis; conflating them leaks.

The new vocabulary (chat / messaging tags + emphasis subset) and the
markdown pipeline are documented in detail in
[message-rendering.md](./message-rendering.md). The implementation
internals live in a sibling `api/mml/` directory (tree parser,
flatten serializer, markdown parser, mention resolvers, entity
helpers, link schemes); **nothing outside `api/mml.ts` may import
from `api/mml/`** — same isolation rule as `api/mql/`.

The private constructor + named factories pattern makes trust explicit:
every site that wraps an untrusted-shape string says `Mml.fromMarkup(...)`,
which is grep-able for security audit. `Mml.compose` is always safe.

`compose` interpolation rules:

- `Mml` fragments emit verbatim.
- Raw strings are escaped (`<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;`,
  `"` → `&quot;`, `'` → `&apos;`).
- Numbers and booleans are coerced via `String()` then escaped.
- Objects with `toMml()` get unwrapped — escaping a non-`Mml` return.
- `null` / `undefined` → empty string.

Vocabulary helpers always re-escape raw string arguments. Calling
`Mml.location("<name>foo</name>")` produces literal text inside
`<location>` tags — devs who want nested markup compose with Mml
fragments explicitly. Pass-through is not a feature; it's the bug
surface this design closes.

**Identity helpers are viewer-aware.** The
`name`/`item`/`object`/`player`/`npc`/`location` helpers don't bake their
display text at compose time — they emit a late-bound fragment whose
inner text resolves at `toString(viewer)` via
`RecognitionApi.describe(viewer, stuff)`, falling back to the viewer-blind
`getPresentation()` when no viewer is threaded. Since `Scene.send`
materializes each frame body against its recipient
(`body.toString(recipient)`), one composed line renders the *right name
per recipient* — "Bob" to a friend, "a hooded figure" to a stranger — for
free. See [belief.md](./belief.md). (`MessageApi.refOf`'s `displayName`,
above, is the one viewer-blind exception: the wire ref is composed before
any recipient is known.)

For prose stored outside the source — schema-declared settings,
CMS-authored room/NPC/item descriptions, eventually prompts — reach
for `ProseApi.format` (see [prose.md](./prose.md)). Liquid-syntax
templates with conditionals and filter chains, same Mml-aware escape
rules as `Mml.compose`, returns a finished `Mml` fragment.

`Mml.escape` is the five-entity escape exposed publicly for other
markup-producers that need it.

`stripTags` parses with a small state machine and decodes the five
built-in entities. Used by clients or log capture that need a
plain-text projection (bare-plain — markdown emphasis drops out).
Tolerates unclosed tags by dropping their characters.

`flatten` runs the same parse but emits per-tag failsafe forms
(emphasis preserved as `**bold**`, lists serialized linear-labeled,
etc.) — for log-capture / archive-export paths that want a
round-tripable markdown projection. See
[message-rendering.md § `Mml.flatten` vs `Mml.stripTags`](./message-rendering.md#mmlflatten-vs-mmlstriptags).

### ⭐ `<quantity>` — a measured value, not a rendered number

Every reading this game computes used to reach the client as
*characters*: `1240 °C` arrived as prose, so a client that wanted to
convert units, compare against a working range, pin the value to an
instrument panel or chart it over time had to re-parse a string it
should never have had to parse. Real quantities are what this game has
that no other text world has, and the markup dropped them at the last
inch.

`Quantity.buildMarkup` emits the structured form. It always carried the
canonical numeric; it now also carries what the reading *means*:

```
<quantity channel="thermal" unit="degC" value="1240" tag="searing"
          via="pyrometer" lo="1150" hi="1300">1240 °C</quantity>
```

| Attribute | Why it is there |
|---|---|
| `channel` | Stable identity for pinning and charting. From `MEASURE_CHANNELS` (`lib/perception/MeasureChannel.ts`). |
| `unit` + `value` | The canonical numeric — conversion, comparison, sorting. |
| `tag` | The qualitative band (`searing`), a different question from `channel`. |
| `via` | **Provenance.** The pedagogical seam: *a reading is only honest if you can say how it was taken.* |
| `lo` / `hi` | Optional working range, declared on the server's authority instead of hardcoded per craft in the client. |
| children | The server's own rendering, so `flatten` and a bare telnet client are unchanged. |

`channel`, `via`, `lo` and `hi` are emitted **only when supplied**, so a
value with no instrument behind it never claims to have one. They ride
an options bag on `toMml`/`formatMml`; `buildMarkup` is the single
chokepoint, which is why ~25 emission sites did not each need editing.

**`via` is derived, not asserted.** `CommandController.affordingSource`
reads `ctx.commandSource` — the affordance layer already knows which
object lit up the verb, so the thermometer that afforded `measure
temperature` *is* the instrument that took the reading. It returns
`undefined` when `commandSource` falls back to the giver, because naming
it there would claim the *person* was the instrument. A new instrument
gets provenance the moment it confers a verb.

#### ⚠ It must never be inert

`quantity` is in `KNOWN_TAGS` and **never** in `INERT_TAGS` or any
passthrough policy, for the reason already written into the inert-tag
list: it is a **factual claim about the world on the server's
authority**, exactly like `speech`. A player who could write it
literally could forge instrument data — worse than misattributing words,
because the whole premise is that the numbers are real, and one forged
reading discredits every honest one beside it. A test asserts it sits on
the same side of the line as `speech`, so the two cannot drift.

Registering it also closed a latent bug: `quantity` was previously in
*no* vocabulary, so `isKnownTag` returned false and by the documented
rule it was a **component candidate** whose name resolves to the module
path `/lib/wiki/components/quantity`.

`flatten` gives it no case of its own — the `default` branch already
returns children verbatim, which is exactly the required failsafe. A
test pins that, so a case added above it cannot silently change the
contract.

#### The channel vocabulary is not the materials `Channel`

`lib/material/Channel.ts` is `edge · point · blunt · shock · heat` — the
shape of a *force*, read by the materials-response fold.
`MEASURE_CHANNELS` is *what kind of reading this is*. **Homonyms at
unrelated layers**: merge them and `heat` means two different things.
Both files say so at the top.

Emitting channels: `thermal` · `mass` · `light` · `atmosphere` ·
`chemistry` · `electrical` · `spatial` · `gravity` · `celestial`.
Reserved (no scalar emitter yet, prose today): `weather` · `time` ·
`response`.

A source-scanning test (`MeasureChannel.totality.test.ts`) enforces the
rule rather than a list: *every scalar-with-a-unit reading emits
`<quantity>` with a channel*. It fails on a measurement verb that does
not exist yet, which is the only version that keeps working — it is what
caught `analyze electrical` shipping conductivity, resistance and
potential through bare `.format()`.

### MarkupAugmenter — inline-affordance pipeline

Authored prose (long descriptions, room narration) often wants
inline MML affordances wrapped around specific keywords: detail
keys become `<detail>` so they're clickable, exit directions
become `<exit>`, named things become `<item>`, untranslatable
text gets masked with a `<unknown-tongue>` wrap, etc. Each of
these is a **text → text transformation** keyed on a contributor
mixin's data; together they form a composable pipeline.

The substrate exposes that pipeline as `MarkupAugmenter`:

```ts
export type MarkupAugmenter = (
  text: string,
  host: Stuff,
  viewer: Stuff,
  opts?: AugmentOpts,
) => string;

static Mml.augment(
  text: string,
  host: Stuff,
  viewer: Stuff,
  opts?: AugmentOpts,
): string;
```

Each contributing mixin declares its augmenters on a static
slot:

```ts
class DetailedMixin {
  static markupAugmenters: MarkupAugmenter[] = [wrapDetailKeysAugmenter];
}
```

`Mml.augment(text, host, viewer)` walks the host's prototype
chain via `MixinApi.getAllMarkupAugmenters` (parent-first → child-
last), folding every contributed augmenter through the text in
order. Pure, sync, viewer-aware (augmenters that don't need the
viewer just ignore it).

**Current customer:** `VisibleMixin.getMarkupLong(viewer)` — the
host-level method that produces the affordance-annotated long
description shipped on every `'detail'` projection. Two live
contributions today: `wrapDetailKeysAugmenter` from `DetailedMixin`,
which wraps canonical detail aliases in `<detail key="...">`
MML, `senseStripAugmenter` on `VisibleMixin` (from the senses
build), which strips regions the viewer's sensorium can't perceive,
and `bulkContentsAugmenter` on `BulkableMixin` (Bulk build), which
appends a holder's contained `Material.appearance` (`look thermos` →
"It holds dark, steaming coffee.").
Future contributors (exit-direction auto-link on
`ExitableMixin`, language masking on a future `LanguageMixin`,
spoiler hide) plug in via the same `static markupAugmenters`
slot with no changes to `Visible` or its consumers.

### ⭐ The identity tags — `actor` / `thing` / `player` / `npc`

Three wire tags, one authoring face. The vocabulary used to have five
(`name`, `item`, `object`, `player`, `npc`); the collapse is the point
of the change, and each half of it was wrong in a different way.

**`item` vs `object` split on portability — which is state, not kind.**
A chair is furniture until somebody picks it up. Nothing downstream
ever branched on the difference, so the split cost two vocabulary
entries and bought nothing. Both are now **`thing`**.

**`name` asserted nothing**, which is exactly why 195 of the tree's
~230 identity emitters reached for it. An emitter holds a `giver` and
cannot know whether a human is on the other end — so it reached for the
tag that let it not say. The wire learned nothing from any of them.

**`Mml.actor` is the face that replaces it**, and it resolves late:

```typescript
Mml.actor(giver)   // the emitter says "a person acting" and stops
// → <player …> | <npc …> | <thing …>, decided at toString(viewer)
```

The decision is `RecognitionApi.kindOf(viewer, target)`, which lives
beside `describe` because it is the same question — *what does THIS
recipient perceive* — asked about the kind instead of the name. It
answers in four steps:

| Gate | Answer | Why |
|---|---|---|
| not an `Organism` | `thing` | Honest failure. Handing an actor face a chair must not promote it to a character. |
| no `HasInteractive` | `npc` | Alive, nobody driving. |
| no viewer | `player` | Logs, snapshots, `refOf`. Nobody is being fooled, and a log that lied would be useless. |
| can't see it, or it's masked | `npc` | ⚠ See below. |
| otherwise | `player` | |

> ⚠⚠ **`<player>` is the sharpest identity claim in the vocabulary.**
> It asserts a real person is behind a figure — precisely the fact a
> disguise exists to hide. So a viewer who cannot see the target, or
> who is looking at a mask, is told `npc`. That is a **false statement
> told to a fooled viewer**, and it is consistent: `getPresentation()`
> already tells that viewer the covering's name. The alternative — a
> third "unknown" tag — would announce *that* something is being
> withheld, which leaks the same fact one level up.

**`actor` is never a wire tag.** It is not in `KNOWN_TAGS`, it never
reaches a parser, a policy, a stylesheet or a client, and a test pins
its absence. Nothing downstream grows a branch for a tag meaning "ask
again."

> ⚠ **Any list of "what is here" must use `Mml.actor`.** Room contents
> include people. `look` splits organisms out to the occupant formatter
> and so was already right; the **sense** verbs do not, and rendered a
> person `<thing>` — Dave read `<thing>` under `sense` and `<npc>` under
> `look`, in the same room, seconds apart. The divergence long predated
> this build and was invisible while both tags meant nothing. Same for
> surface-resting lists (someone sits on a stool) and search results
> (what a search turns up is very often a hiding person). Only a
> definitionally-object list — inventory — stays `Mml.thing`.

**Use `player` / `npc` / `thing` directly only where the emitter knows
something the world does not** — the puppeteer behind a possessed
corpse, an illusion that should read as a person, a corpse being named
as an object. Those are the cases the framework must not guess at, and
they stay explicit. Everything else says `actor`.

All three wire tags are **identity claims**, so no passthrough policy
below `all` admits them — a player who could write `<player>` could out
somebody, and one who could write `<thing>` could plant a clickable
pointing at an object that is not there.

## Scene — multi-audience composer

`MessageApi.scene(actor)` returns a `Scene` builder. Layer 3 of the
composer stack: producers describe per-audience frames; one `.send()`
dispatches them all.

```typescript
MessageApi.scene(speaker)
  .topic('speech.vocal')
  .toSelf(Mml.compose`You say, ${Mml.speech(text)}`)
  .toPeers(Mml.compose`${Mml.actor(speaker)} says, ${Mml.speech(text)}`)
  .payload({ speaker: MessageApi.refOf(speaker), text })
  .send();
```

Surface:

- `MessageApi.scene(actor)` — establishes actor identity. Actor is any
  `Stuff`. Compositional requirements (Sensor, Containable, Container)
  are checked per `.toX()` method, not at the factory.
- `.topic(path)` — required.
- `.modality(name)` — optional. Stamps the perception-modality
  attribution on every composed frame's `meta.modality`. Reception
  gating at `SensorMixin.filterMessage` drops the frame for
  recipients whose sensorium doesn't include this modality (except
  actor self-frames, which always deliver). Sensory producers
  populate it: `VocalMixin.say` → `'hearing'`, `AetherMixin.tell`
  → `'verbal-esp'`. System / log / narrative frames omit it and
  deliver unconditionally. See [senses.md](./senses.md) for the
  cross-modality substrate.
- `.tags(tags[])` — optional shared tags merged with per-audience auto-tags.
- `.payload(p)` — optional shared payload, used for any audience that
  doesn't override.
- `.toSelf(body, payload?)` — actor frame. Auto-tags `audience:actor`.
  Throws if the actor isn't a Sensor.
- `.toTarget(target, body, payload?)` — single-target frame. Auto-tags
  `audience:target`. Throws if the target isn't a Sensor.
- `.toPeers(body, payload?)` — broadcasts to actor's environment;
  excludes actor and any explicit target. Auto-tags `audience:witness`.
  Throws if the actor isn't Containable.
- `.toContents(body, payload?)` — broadcasts to actor's contents;
  excludes the actor. Auto-tags `audience:witness`. Throws if the actor
  isn't a Container. Used when the actor IS the container — a haunted
  location speaking to its occupants.
- `.toAudible(body, { descriptor })` — discrete-event cross-room sound
  push (attenuated + directional, closed doors block; reads
  `meta.acousticDb`). See
  [perception.md § Discrete-event sound push](./perception.md#discrete-event-sound-push--audible--scenetoaudible).
- `.send()` — dispatches all configured frames.

Per-audience compositional requirements throwing is deliberate. They
are composition errors, not user-input errors — a controller that asks
for `.toPeers` on a non-Containable actor has a bug, not a recoverable
condition.

The Scene class lives in `mud/lib/message/Scene.ts` (relocated there in
the surface-architecture refactor's DP.1 — value classes live in `lib/`,
not `api/`) and is re-exported from `mud/api/message.ts`, so callers
still reach it through the face. Construction goes through
`MessageApi.scene(actor)` only; Scene's constructor is sentinel-gated
and rejects all other callers.

Constraints enforced inside the builder:

- Only one frame of each kind per Scene. Multiple `.toSelf()` calls
  throw — one frame per audience role per scene.
- `.toPeers()` exclusion: when both `.toSelf()` and `.toTarget()` are
  present, `.toPeers()` excludes both — witnesses are everyone except
  the actor and target.

`.send()` does the per-frame plumbing:

- Reads the ambient `ExecutionContextApi` for `commandId` and
  `causingCommandId` and stamps them onto every frame's `meta`.
- Builds one `MessageFrame` per queued audience: tags include the
  per-audience `audience:*` tag plus shared tags; body is the `Mml`
  fragment's `toString()`; payload is the per-audience payload, falling
  back to the shared payload.
- For self/target: dispatches via `MessageApi.sendMessage(recipient, frame)`.
- For peers: walks `MessageApi.getSensors(env)` and dispatches one frame
  per sensor, skipping the actor and the explicit target.
- For contents: dispatches via `MessageApi.messageContents(actor, frame,
  { exclude: actor })`.

## VocalMixin and the Containable-wins rule

`VocalMixin.say(text)` is Layer 4 sugar — the canonical "speak in your
current scope" pattern. Speakers get `say` for free as long as they
compose the mixin. The user-supplied text runs through
`Mml.markdownToMml` (Discord-dialect subset; see
[message-rendering.md](./message-rendering.md)) with the speaker's
perceiver mention resolver before being wrapped in `<speech>`.

Scope is inferred from the speaker's composition (the **vessels rule**):

```typescript
if (MixinApi.isContainable(speaker))    scene.toPeers(witnessBody);
else if (MixinApi.isContainer(speaker)) scene.toContents(witnessBody);
else throw new Error(
  'VocalMixin requires composition with Container or Containable'
);
```

A vessel is a Stuff that's both Container *and* Containable
(wardrobe, cart, walking house). The Containable path wins: vessels
broadcast to peers (other things in the same area), not to contents.
A speaker that's neither — composition error — throws.

`MobileMixin` follows the same Containable-wins rule for movement
narration; its base-class constraint
(`TBase extends MixinConstructor<Stuff & Containable>`) makes the
guarantee compile-time.

### AetherMixin — the non-acoustic sibling

`AetherMixin` is the parallel capability mixin for non-acoustic
transport — `tell` today, future channel `chat` and remote `emote`.
The Aether is the in-fiction comm-network (implant network, magical
resonance field, hybrid mesh — diegetically per zone). A character
can compose Vocal without Aether (mute) or Aether without Vocal
(post-vocal-loss with an implant); both are independent.

`AetherMixin.tell(target, text)` fires `speech.comms` with
chat-form bodies (`<speaker> → <target>: <body>` self,
`<speaker> → you: <body>` target). Self is a valid target. Markdown
parsing runs through `Mml.markdownToMml` with the speaker's perceiver
mention resolver, same as `VocalMixin.say`. `tell` accepts either a
single `Stuff` or a `readonly Stuff[]` — multi-target stamps
`payload.recipients: StuffRef[]` + (when chat-backed) the cohort's
`meta.channelId`, and the per-target frame includes a
`"(also to: …)"` cohort hint so each recipient sees who else was
addressed. Cohort tracking lives on AetherMixin itself as runtime-
only state: `getLastInboundCohort()` / `getLastOutboundCohort()`
power the `reply` and `dm .` pronoun verbs.

Avatar composes `AetherMixin` because players have implants per the
char-gen / augmentation slates' diegetic story. NPCs opt in
per-class when content requires it. See
[message-rendering.md § AetherMixin](./message-rendering.md#aethermixin--non-acoustic-transport).

### VocalMixin acoustic family — say / whisper / shout / `--to`

`VocalMixin` extends to three acoustic verbs sharing one composition
backbone: `say(text, target?)`, `whisper(text, target?)`,
`shout(text, target?)`. Each stamps `meta.modality = 'hearing'` and
a per-verb `meta.acousticDb` source amplitude (30 / 60 / 90 dB
respectively) for the sound-propagation walk's reach computation.
With `--to <target>` (a flag on `say.yaml` / `shout.yaml`), the
room still hears, but the target gets a marked target-frame
("Bobalu says to you, …"); whisper is implicitly directed so its
`target` is a required positional. Topics: `speech.vocal`,
`speech.quiet`, `speech.vocal`.

### SoulMixin — expressive (emote) on every Character

`SoulMixin` is parallel to `VocalMixin` for the emote channel: the
ESP-modality expressive surface (waves, smiles, bows, free-form
prose). NOT augment-gated — composed natively on every
`Character`; the diegetic reading is that expression itself is
innate, with the carrier signal (the Aether) augment-gated
separately for remote delivery. The mixin owns rendering
(`renderEmote` / `renderFreeForm` return per-audience Mml triples)
AND in-room send (`emote` / `emoteFree` compose Scene + send,
Containable-wins). Channel-routed / DM-handle-routed emotes call
the render methods directly and compose their own Scene. Topic:
`act.emote`. Modality stamp: `'emotive-esp'`. See
[emotes.md](./emotes.md) for the full substrate (Emote Documents,
EmoteGrammar, SoulCatalogue, dispatch paths).

## Routing pipeline

`MessageApi.sendMessage(recipient, frame)` is the **lone delivery
chokepoint**. Every routing helper, every `Scene.send` dispatch, and
every `MudlogApi` emit goes through it. Non-MessageApi code does NOT
call `sensor.onMessage` directly — the chokepoint is where future
cross-cutting concerns (audit trail, debug logging, bus-level taps,
wire-level filters) hook in exactly once.

```typescript
static sendMessage(recipient: SensorStuff, frame: MessageFrame): void {
  recipient.onMessage(frame);
}
```

The two low-level routing helpers complement it:

- `MessageApi.getSensors(container)` — walks `container.getContents()`
  and filters via `MixinApi.isSensor()`. Mixin-narrowed; nothing
  duck-types `onMessage`.
- `MessageApi.messageContents(container, frame, { exclude? })` — deliver
  to every sensor inside the given container. The caller picked the
  container. Used when the speaker IS the container.
- `MessageApi.messageContainer(source, frame, { exclude? })` —
  convenience wrapper for the "speaker is Containable, broadcast to peers
  in the same room" case. Drops with a warning if `source` has no
  environment.

`Avatar.handleMessage` is the multiplexing fan-out:

```typescript
protected override handleMessage(frame: MessageFrame): void {
  const app = Avatar.getApplicationInstance();
  for (const interactive of this.interactives) {
    app.sendMessageToInteractive(interactive, frame);
  }
}
```

`Application.sendMessageToInteractive(interactive, frame)` is the sole
gateway from game objects to Backend. It calls
`Backend.sendMessageToSocket(socketId, frame)`, which `JSON.stringify`s
and writes to the WebSocket. For non-Avatar Sensors (NPCs, custom
observers), `handleMessage` does whatever that Sensor does — there's no
special path.

## Routing rules — retained, not currently driving placement

A **rule is a filter with an address**: an ordered table, first match
wins, each rule a predicate over the envelope
(`address` / `actor` / `weight` / `topic` prefix) plus a destination and
a MOVE-or-COPY disposition. `Avatar.handleMessage` still evaluates it
and stamps `meta.feeds` at the delivery seam.

⚠⚠ **Nothing reads that stamp any more.** The client's tabs are
predicates over the whole buffer, not destinations a rule placed
something in — see
[client-shell.md](./client-shell.md). Two properties retired the
destination model and neither was patchable within it:

- the stamp is applied at **delivery**, so changing a rule never
  re-sorts history; and
- the frame store does not persist it, so backfilled frames carry none
  and fall to the catch-all. Measured live, that put a player's entire
  dm history outside their Attention feed.

The rules are kept — `DEFAULT_ROUTING`, `MessageApi.feedsFor`, the
undeletable catch-all and their tests — because *which frames should
ping you* is a real question that wants exactly this shape, and it is
the next thing this substrate should drive. Recorded here so the stamp
is a deliberate holding pattern rather than a mystery.

⚠ **Read `NotifyPolicy` / `NotifyRule` before designing anything on top
of it.** A notification model already exists in the social graph
([social-graph.md](./social-graph.md)); a second one grown out of these
rules would be two answers to *what should reach me*, with the weaker
one in the way. The bell has been cut twice for that reason. Whichever
survives, this predicate vocabulary is the shape the rules want — not a
new one.

⚠ The catch-all rule remains undeletable in the model. Every frame must
land somewhere, and in a world where a frame can be *you are on fire*, a
lost message is not a cosmetic bug — that invariant should survive
whatever consumes the rules next.

## Sensor extension points

`SensorMixin` is shaped as a template method — `onMessage` is
framework plumbing, content authors hook the named extension points:

```typescript
onMessage(frame: MessageFrame): void {
  const transformed = this.filterMessage(frame);
  if (transformed === null) return;       // dropped
  this.handleMessage(transformed);
}

protected filterMessage(frame: MessageFrame): MessageFrame | null {
  // Built-in modality-attribution check (2026-06 perception build):
  // drops frames whose `meta.modality` organ key isn't in the
  // recipient's PerceptionApi.sensorium. Actor self-frames bypass
  // (audience:actor tag). System / log / narrative frames omit
  // meta.modality and deliver unconditionally.
  // ...shadows further extend this contract.
  return frame;                            // shadowable
}

protected handleMessage(_frame: MessageFrame): void {
  // subclass override (Avatar, NPCs, …)
}
```

- **`filterMessage`** is the per-recipient interception point. Game
  content (deafness shadows, audit observers, language garblers) shadows
  this method via the shadow framework. Returning `null` drops the
  frame; returning a transformed frame substitutes it. Lifecycle
  binding, ordering, and call-security mediation are inherited from
  `ShadowApi`. Hooks SHOULD treat frames as immutable and return new
  objects when modifying:

  ```typescript
  return { ...frame, body: garble(frame.body) };
  ```

- **`handleMessage`** is the subclass-override delivery point. Avatar
  multiplexes to its connected Interactives. NPCs run AI. Default is a
  no-op.

`onMessage` itself is NOT meant to be shadowed or overridden — it's the
contract ("sensors receive messages"). Shadowing it would let content
skip `handleMessage` and break delivery. Game content uses
`filterMessage`; subclasses use `handleMessage`.

## MudlogApi

`MudlogApi` (`mud/api/mudlog.ts`) is the in-game messaging facility for
log-style content. Every call delivers to a Sensor. Topic is
`shell.diagnostic.<level>` (no category) or `shell.diagnostic.<category>.<level>`.

```typescript
class MudlogApi {
  // Body-only — topic shell.diagnostic.<level>
  static trace(body: Mml, opts?: MudlogOptions): void;
  static debug(body: Mml, opts?: MudlogOptions): void;
  static info (body: Mml, opts?: MudlogOptions): void;
  static warn (body: Mml, opts?: MudlogOptions): void;
  static error(body: Mml, opts?: MudlogOptions): void;
  static fatal(body: Mml, opts?: MudlogOptions): void;

  // Categorized — topic shell.diagnostic.<category>.<level>
  static trace(category: string, body: Mml, opts?: MudlogOptions): void;
  // …debug/info/warn/error/fatal same shape

  // Hook for log4j-style "skip composition when disabled" guards.
  // Always returns true today; per-category levels and dynamic config
  // plug in here.
  static isEnabled(category: string | undefined, level: LogLevel): boolean;
}

interface MudlogOptions {
  to?: SensorStuff | SensorStuff[];   // explicit recipient(s)
  payload?: unknown;
}
```

`SensorStuff = Stuff & Sensor` — the option is mixin-narrowed, not a
loose `Sensor` interface.

Recipient resolution:

1. If `opts.to` is provided → deliver to those Sensor(s).
2. Else look up the active command giver via
   `ExecutionContextApi.getCurrentCommandContext()?.commandGiver` and
   check via `MixinApi.isSensor`. If present → deliver to them.
3. Else **throw**. There is no stdout fallback. MudlogApi is purely an
   in-game messaging facility; stdout / file logging is a separate
   concern that has nothing to do with MudlogApi.

Every `error()` and every other level throws on no recipient, same
rule. That's the `MudlogApi` contract: "calling without an obtainable
recipient is a bug."

Each emit attaches `level:<level>` to `tags`, plus `category:<category>`
when the categorized overload was used. Frames also carry `meta.commandId`
and `meta.causingCommandId` from the ambient `ExecutionContextApi` — same
auto-stamp `Scene.send` performs.

```typescript
MudlogApi.info('hot-reload',
  Mml.compose`Reloaded module ${moduleRef}, ${dependents.length} dependents`);

MudlogApi.info(Mml.compose`Boot complete`);   // body-only

if (MudlogApi.isEnabled('combat', 'debug')) {
  MudlogApi.debug('combat',
    Mml.compose`${Mml.actor(attacker)} swing detail: ${expensive.computation()}`);
}

MudlogApi.info('admin',
  Mml.compose`${Mml.actor(player)} just hit level 50`,
  { to: [admin1, admin2] });
```

The standard level constants are exported as `MUDLOG_LEVELS` for
diagnostic UIs that want to render every level.

### Input echo at `shell.diagnostic.*`

`CommandGiverMixin.executeCommand` fires an **input-echo** MessageFrame
at the *start* of every dispatch — before parsing, matching, or
controller execution. The frame carries topic
`shell.diagnostic.info` when the parser succeeded or
`shell.diagnostic.warn` when it failed; payload is:

```typescript
{
  kind: 'issued';
  rawText: string;
  expandedText?: string;     // present when alias expansion fired
  verb?: string;
  parseError?: string;       // present iff parse failed
  dispatchId: string;        // = the originating commandId
  originInteractiveId?: string;  // absent for programmatic dispatch
}
```

The frame is multiplexed via `Avatar.handleMessage` to every connected
Interactive on the actor. Use cases:

- **Multi-device echo** — an Avatar's other Interactives see what
  their sibling typed; the client filters its own echo by comparing
  `payload.originInteractiveId` against the `interactiveStuffId` it
  stashed from `session.link`.
- **Audit trail** — server-side audit Sensors observe player input
  independently of any dispatch outcome.
- **Replay capture** — structured `kind: 'issued'` records replay
  cleanly.

The echo is the entry-time signal; the dispatch-response envelope is
the outcome carrier. Both share `dispatchId` (= the originating
`commandId`) for correlation. See
[response-envelope.md](./response-envelope.md) for the envelope shape
and the wire-correlation table.

## Movement-message defaults — `MobileMixin.settings`

Default movement / teleport prose lives as schema-declared settings
on `MobileMixin` (`messages.movement.*`). Avatars compose
`EnvironmentMixin` and players override individual templates through
the `settings` command; NPCs and other Mobile hosts that don't
compose `EnvironmentMixin` render at the schema default via
`resolveSetting`'s non-Environment fallback. See
[shell-environment.md](./shell-environment.md) for the broader
settings subsystem.

The override hierarchy `MobileMixin` consults (highest priority first):

1. **`Exit.messageOut` / `Exit.messageIn`** — room-author per-exit
   Liquid templates with `{{ mover }}` available. Handy for one-off
   custom narration on a specific door.
2. **Room hooks** on the source/destination Container:
   `getDepartureMessage(mover, exit)`, `getArrivalMessage(mover, exit)`,
   `getTeleportOutMessage(mover)`, `getTeleportInMessage(mover)` —
   each returns `{ self?, peers? }` and Mobile fills in defaults for
   any audience the resolver skipped.
3. **`messages.movement.*` settings** — schema-defaulted, player-overridable.

Each schema entry's default is rendered through `ProseApi.format`,
which interpolates `Mml.actor(mover)` and `Mml.direction(...)` and
handles the directional/bland arrive split via `{% if direction %}`
inside a single template — see [prose.md](./prose.md) for the
templating language.

Out of scope for these settings: prose for one-shot controller output
(look, inventory, get/drop, open/close, etc.) lives where the controller
composes it. Most controller prose has no use for configuration —
inline `Mml.compose` is fine.

## Command attribution

Every frame composed during a command's *synchronous* execution carries
both `meta.commandId` and `meta.causingCommandId` (same value). Frames
composed in async aftermath of a command (via a propagating
`ScheduleApi` callback) carry `meta.causingCommandId` but not
`meta.commandId`. Frames composed outside any command lineage carry
neither.

| Scenario                              | commandId | causingCommandId |
|---------------------------------------|-----------|------------------|
| Sync command execution                | set       | set (same id)    |
| Async aftermath of a command          | absent    | set              |
| Pure background (NPC AI, tick)        | absent    | absent           |

`CommandContext` already exists as the per-command bundle
(`commandGiver`, `interactive`, `location`, `commandText`, `executionId`)
and carries `commandId: string` for attribution.
`CommandGiverMixin.executeCommand` is the home of the lifecycle:

1. Generates a fresh `commandId` (nanoid) per execution.
2. Builds the CommandContext, stamping `commandId`.
3. Pushes the CommandContext onto `ExecutionContextApi.commandContext`
   AND sets `causingCommandId` to the same id.
4. Emits the input-echo MessageFrame (see "Input echo at
   `shell.diagnostic.*`" above) and the dispatch-response envelope
   carries the outcome.
5. Invokes the controller.
6. Pops on return (including error paths). `causingCommandId` clears
   unless propagated by a scheduler.

Inside the controller, ANY frame composed via `Scene.send()` or
`MudlogApi.*` — directly, via mixin sugar, or transitively via deeper
calls — reads `ExecutionContextApi` at compose time and auto-stamps
`meta.commandId` / `meta.causingCommandId`. Producers don't pass
attribution explicitly.

### ScheduleApi — async propagation

`ScheduleApi` (`mud/api/schedule.ts`) is the framework's scheduler.
Single-purpose for v1: one-shot, recurring, cancel.

```typescript
class ScheduleApi {
  static schedule(delayMs: number, fn: () => void,
                  opts?: ScheduleOptions): ScheduleHandle;
  static recurring(intervalMs: number, fn: () => void,
                   opts?: RecurringOptions): ScheduleHandle;
  static cancel(handle: ScheduleHandle): void;
}

interface ScheduleOptions {
  // Default true. Captures
  // ExecutionContextApi.getCurrentCausingCommandId() at schedule time
  // and re-plants it on a fresh Root frame inside the callback.
  // Pass false to sever the chain.
  propagateAttribution?: boolean;
}

interface RecurringOptions extends ScheduleOptions {
  initialDelayMs?: number;          // default = intervalMs
  mode?: 'fixed-rate' | 'fixed-delay';   // default 'fixed-delay'
}
```

`recurring` defaults to `fixed-delay` (next fire = previous-completion
+ interval, drift-tolerant); `fixed-rate` uses `setInterval`'s
predictable cadence (long callbacks can pile up).

Propagation mechanics: when a callback fires, `ScheduleApi` runs it
inside `ExecutionContextApi.runRoot(ScheduleApi, 'fire', …)` — a fresh
Root frame with no live CommandContext, but the captured
`causingCommandId` is re-planted onto the frame's metadata via
`updateCurrentFrameMetadata`. Frames composed inside the callback then
see `causingCommandId` (the originating command) but no `commandId` (no
live execution).

Cascading work extends naturally. If the async callback itself calls
`ScheduleApi.schedule` with propagation, the chain extends as long as
schedulers keep propagating. There's no built-in depth limit;
`propagateAttribution: false` severs the chain.

For recurring schedules, attribution is captured once at schedule time
and reused for every fire. If the originating command is "lighting a
candle that ticks once a second," every tick reads back the same
`causingCommandId`.

## Design rationale

### Composition model (server-side, per-audience)

Messages are composed once on the server, by the producer, with both
their structured payload and console body bound at compose time. The
wire carries both. The alternative — domain events with edge-side
renderer registries — was rejected because it requires either
client-side rendering work (out of scope) or server-edge renderer
registries, which introduce a registry-rot failure mode and break
locality for AI-agent authors.

When the actor and witnesses see different bodies of the same event,
the producer composes both at the same call site (one Scene, multiple
`.toX()` calls). The wire carries one frame per audience.

### Topic vs tags vs scope

Multi-axis classification doesn't fit a single hierarchy. Topic encodes
the producing subsystem (speech machinery → `speech.*`); tags
capture orthogonal properties (`audience:witness`); scope is a
producer-side concern that disappears once the frame lands.

Two roots — `world.*` and `system.*` — and that's it. Logs are
`shell.diagnostic.*`. Widgets are NOT a topic root; widgets are renderers
that consume topic streams and are registered on the client side.

### Sensors are the only recipient type

No "sinks", no "channels", no parallel delivery mechanisms. Anyone
wanting to consume messages (in-game admin console, replay capture,
audit observer) implements or shadows a Sensor. One mechanism, one
lifecycle, one security model.

### MudlogApi is in-game, not stdout

Every MudlogApi call delivers to a Sensor. There is NO stdout fallback,
no singleton sensor, no "framework log" concept. Stdout logging is an
entirely separate concern handled by `console.log` or a future framework
facility. Conflating the two would force every MudlogApi call site to
think about both the in-game and out-of-game audience.

### Shadowing rather than a hook registry

`SensorMixin` exposes `filterMessage` as a shadowable extension point
because lifecycle binding, ordering, security mediation, and call-stack
semantics are already provided by the shadow framework. A parallel hook
registry would re-implement all of these. Single mechanism for content
authors to learn.

`onMessage` itself isn't shadowable: it's the contract. Shadowing it
would let content skip `handleMessage` and break delivery.
`filterMessage` is the explicitly-shadowable surface; `onMessage` and
`handleMessage` are framework plumbing.

### No global heartbeat — periodic behavior is explicitly scheduled

The framework does NOT provide a global tick that objects subscribe to.
Every recurring behavior schedules itself via `ScheduleApi.recurring`
with its own interval. Reasons:

- Cost scales with active scheduled tasks, not with object count —
  10,000 inert objects burn zero CPU.
- Cadence is legible at the call site (grep `schedule` / `recurring`
  reveals every periodic behavior and its interval).
- Fine granularity per task instead of a global compromise interval.
- Matches Node.js's event-loop model rather than fighting it.

Future "reset" mechanics, time-of-day cycles, etc. layer on top of
`ScheduleApi`, not via a heartbeat.

### Containable wins for vessels

When an actor implements both Container and Containable (a vessel —
wardrobe, cart, walking house), the Containable path wins. Vessels
broadcast to peers (other things in the same area), not to contents.
The room a wardrobe sits in is a more meaningful audience for the
wardrobe's speech than the things stuffed inside it. Mixin sugar
checks Containable first; pure-Container actors (a haunted location)
fall through to `toContents`.

## The renown reception seam (`onMessage` + `MessageApi.isCommunicative`)

`SensorMixin.onMessage` is the receive-side hook the renown **reception**
generator taps: after `filterMessage` passes (so it's perception-gated),
a *communication* command frame fires `CommReceivedEvent` to credit the
speaker with a small being-heard signal. The gate is
`MessageApi.isCommunicative(topic)` (forwarding to `MessageLogic` →
`TopicCatalogue`'s `communicative` set), so only say/whisper/shout/emote/
chat frames — not dm/narration/system — reach the bus. Renown recovers
the speaker from the act registry (`ReactionApi.actInfo`). See
[renown.md](./renown.md).

## Cross-References

- [message-rendering.md](./message-rendering.md) — full MML vocabulary
  including the chat/messaging tags + emphasis subset, the markdown
  pipeline, `AetherMixin`, client-side renderer, stylesheet engine,
  themes, per-message-type templates, the `style` verb, and the
  `client-state-update` push wire
- [topics.md](./topics.md) — `TopicCatalogue` (authored topic
  descriptors; the source of truth after the `MessageApi.Topics`
  constant tree retirement)
- [templates.md](./templates.md) — template clone pipeline that creates
  the Stuff that compose `SensorMixin`, `VocalMixin`, `MobileMixin`
- [lifecycle.md](./lifecycle.md) — Stuff create/destroy choreography;
  Avatar's `onDestruct` clears its `interactives` set
- [persistence.md](./persistence.md) — `Document` / around-hook
  machinery; unrelated to messaging but shares `PersistenceManager`
- [state-model.md](./state-model.md) — Avatar self-contained design;
  Avatar runtime fields (`interactives`) are not persisted
- [antipatterns.md § Duck Typing with Mixins](../antipatterns.md#duck-typing-with-mixins)
  — why `MessageApi.getSensors` narrows via `MixinApi.isSensor` rather
  than checking for an `onMessage` method
