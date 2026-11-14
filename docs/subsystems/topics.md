# Topics substrate

The substrate for "what kind of message is this?" — per-topic authored
descriptors plus a runtime catalogue served to the client at
session-establish.

Topics carry the routing decision on every `MessageFrame`: the
wire-level `topic` string is a dotted path like `speech.vocal`,
and the catalogue resolves each path to a friendly label, a
description, a family, and six facets. The cockpit's tabbed terminal,
filter drawer, and gutter tooltips all consume that resolution; the
future help system will tap the same catalogue.

## Why the filter lives on the client

**The server prints everything; the client decides what to show.** The
server's job is to emit cleanly categorized frames with a stable topic
and its facets; the client's job is to give the player tools to slice
that stream however suits them. Always-print on the server,
always-categorize on the wire, always-filterable on the client: the same
prose is available to anything that wants it (audit trails, replay, log
capture), and a player's session view is only a filter over the
firehose. Filtering is cheap precisely because it lives there — no
round-trip, no policy negotiation, instant feedback.

The shipped shape confirms it rather than merely permitting it: a
cockpit tab is a named predicate evaluated at render over the whole
buffer ([client-shell.md § One strip, and every tab is a VIEW over the
whole buffer](./client-shell.md)), and a rule change re-sorts history
only because the server never withheld a frame in the first place. The
delivery-time routing stamp that was retired failed on exactly this
point. *(Graduated from the console-filtering slate, 2026-09.)*

## ⭐⭐ The tree carries subject matter; the facets carry everything else

**The single governing rule.** A topic path answers exactly one
question — *what is this frame about?* Every cross-cutting question —
who it is aimed at, whose voice it speaks in, how much it matters,
which surface it belongs to — is a **facet**, never a path segment.

This is worth stating loudly because the vocabulary violated it for
years and the damage was invisible. The pre-facet tree had **89 topics**
because it was five facets flattened into a string:

| The tree encoded | The facet that carries it |
|---|---|
| `system.*` vs `world.*` | `actor` |
| `say` / `whisper` / `shout` | `address` |
| `dm` | `address: direct` |
| 13 × `system.shell.*` | `actor: system` + `address: direct` |
| `*.error`, `system.log.*` | `weight: diagnostic` |

Once the facets existed the tree had nothing left to do but subject
matter, and it collapsed to **seven roots and 29 leaves** with nothing
lost. Two collapses came free: the 15 `world.perception.measurement.*`
leaves became one `sense.reading`, because `<quantity channel="thermal">`
already carries the channel in the markup; and the three per-platform
relay topics became one `speech.relay`, because the platform is a
transport attribute, not a subject.

**The test to apply before adding a topic:** if the thing that
distinguishes your candidate from an existing leaf is *who is talking*,
*who it is for*, or *how loud it is*, it is a facet value, not a topic.

### The seven roots

| Root | Leaves | Subject |
|---|---|---|
| `speech` | `vocal` `comms` `channel` `relay` | Words from a person, split by **medium** — overhearable vs point-to-point is a fact about the world |
| `act` | `deed` `move` `emote` `combat` | Something was done; `actor` distinguishes person from world |
| `sense` | `ambient` `weather` `survey` `reading` | The world reaching you; `survey` answers your own looking (the topic key — `look`, `search`, `inventory`; there is no `survey` verb since controller-liquidity W17) |
| `self` | `body` `standing` `holding` `group` | Your own person |
| `publication` | `press` `wiki` `forum` | Authored durable content |
| `shell` | `result` `error` `prompt` `config` `control` `diagnostic` | The client↔server relationship |
| `session` | `link` `identity` `notice` `presence` | The connection itself |

### ⚠ The vocabulary is open; the roots are closed

Content packs **may** add topics — they ship `topic` documents under
`content/topics/` exactly as the platform pack does, and the dictionary
reads every document of the kind, not a bundled file list. Nothing
special is needed.

**But a pack may add leaves only, never a root** (the installer's
`validatePackTopics`, over the pack's topic documents, throws before any
write). If a pack could mint a
top-level subject, a player's mute of `sense` would stop catching
everything sense-shaped — and subtree-mute integrity is the entire
reason topics form a tree instead of a flat tag set. `TOPIC_ROOTS` in
`@saxonberg/types` is the closed list, exported for the same reason as
`LAYOUT_NAMES`: the server's gate and the client's filter surface must
never drift.

The convention for content, in order of preference:

1. **Use an existing leaf** and distinguish with facets. This is almost
   always right.
2. **Add a leaf** under a core root, when the *subject* genuinely
   differs from every existing one.
3. **Never a root.**

The rule was written against a live violation: `residence.provision` /
`.remodel` / `.unprovision`, minted as a third root by Duncan Hall's
domain-local commands. All three are a person doing something
observable — `act.deed`, with `actor` carrying the distinction. Nothing
was lost by not having a root, which is the evidence the rule is right
rather than merely tidy. The two other content-local families
(`world.lounge.*`, `world.hazard.*`) collapsed the same way.

### ⭐ The two-part totality gate

Tier 3 (below) *derives* a descriptor for any key it has never heard of,
so a typo, a rename, or a never-authored topic used to fail **silently**
into something plausible. Measured when the gate was written: **105
topics were emitted, 89 were seeded, and 45 emitted topics had no
authored descriptor at all.** Nobody had noticed, because a derived
default reads like a real one.

A build-time lint alone cannot close this, because it cannot see a
third-party pack. So the gate has two halves:

- **`pnpm lint:topics`** (`scripts/check-topic-keys.ts`, CI-gating) —
  every topic key emitted in server source resolves to an authored
  descriptor, and every key's root is in `TOPIC_ROOTS`.
- **Runtime** — a topic that resolves by *derivation* rather than
  authorship files a diagnostic through `DiagnosticApi`, once per key.
  Resolution is unchanged; the frame still renders. The failure becomes
  author-visible instead of invisible.

⚠ **The lint has no exemption list, by design.** All five ways
`.topic(…)` is written in this tree resolve statically — literals,
`*_TOPIC` constants, class fields with concrete initializers, local
consts holding literal ternaries, and forwarded option properties — so
an unresolvable argument is an **error**, not a skip. A silently-skipped
emitter is precisely how the 45 stayed hidden.

⚠⚠ **Resolution is file-scoped first.** An early revision of the lint
resolved names against one tree-wide table, and `Mobile.ts`'s
`.topic(topic)` matched an unrelated file's `topic` default — resolving
to a plausible *wrong* key rather than reporting a miss. The gate went
green while two emitters kept firing retired topics. A name-based
resolver must prefer the local declaration or it manufactures the exact
quiet wrongness it exists to catch.

The lint also **warns** (never errors) on a seeded key nothing emits.
That direction produced ~50 dead seeds in the old corpus. It cannot be
an error — packs and subtree parents legitimately have no core emitter —
but it must not be silent either.

## File layout

| File | Role |
|---|---|
| `lib/message/Topic.ts` | The value class owning the dictionary: `warm`, `warming`, `install`, `invalidate`, `descriptor`, `snapshot`, `isCommunicative`. Static, never instanced; no catalogue singleton. |
| `content/platform/content/topics/<dotted.key>.yaml` | One `topic` document per leaf or family, stored at `/platform/topics/<dotted.key>`. Flat file names, no nested directories; the dotted key is the path's leaf. |
| `@saxonberg/types` `TopicDescriptor` | Wire-safe shape: `{ topic, family, label, description }` + the six facets. `TopicRoot` / `TOPIC_ROOTS` live here too. Shared between the server snapshot and the client cache. |

Nothing is on any pack's `boot:` list: the dictionary is never booted
(below).

## No code-side constants mirror

There is no `TOPICS` const in code. The earlier nested `TOPICS` tree
on `MessageApi` has been retired — topics are dotted-path string
literals at call sites (e.g. `.topic('speech.vocal')`), with the
`topic` documents as the runtime vocabulary. Three deliberate
non-things:

- No `Topics` enum exported from `@saxonberg/types`.
- No mirror file under `lib/topics/` (and no such subsystem dir —
  topics are a messaging concern, hence `lib/message/`).
- Topic strings cross the wire raw; the descriptor carries everything
  beyond the string itself (label, description, family, facets).

## A topic is a document

A topic is a `topic` document ([document-store.md](./document-store.md))
— authored metadata nothing writes and nothing acts on. It was a
`Topic` Idea under `/platform/idea/Topic/`, read by a booted
`TopicCatalogue` singleton, until the documents path identity build
(2026-11); both are gone.

```yaml
# packages/content/platform/content/topics/act.combat.yaml
family: "act"
label: "Fighting"
description: "Blows, gambits, and the turns of a fight."
address: personal
actor: person
weight: consequence
audience: player
durable: true
affordance: decays
```

The file IS the data: no `class:`, no `data:` wrapper, and no `topic:`
line — the dotted key is the path's leaf (`Topic.keyOf`). `family` is a
stored string rather than derived from the dotted path, so authoring can
override the inheritance chain; the fallback resolver uses the dotted
structure directly. `topic` declares its key as a typed word, so two
packs claiming one key are refused at install, naming both
([content-packs.md](./content-packs.md)). Authors edit descriptors
through the workspace shell or the CMS — there is no `describe-topic`
verb.

## ⭐ The dictionary loads on the first scene (the one engine-wired load)

A topic is **named by code** (`.topic('combat.hit')`, ~1,200 emitters) and
cited by no holder, so no host's setter or `onCreate` can carry its load
the way every other document family's is carried. The message pipeline is
engine machinery, so it loads the vocabulary itself — **transparently,
with no caller involvement and no frame lost** (documents D13; the owner:
*"queue is fine"*):

- `MessageApi.topics().warm()` reads every `topic` document (one `listOfKind`, memoized)
  and installs the dictionary. Nothing calls it at boot.
- **`Scene.send()` stays synchronous for its caller.** While the
  dictionary is cold (`MessageApi.topicsWarming()` returns the warm in
  flight), dispatch is pushed onto an ordered queue behind that one warm,
  and every queued scene dispatches, **in order**, when it lands. A scene
  sent after the warm dispatches immediately — and a scene sent while the
  queue still holds earlier ones queues behind them, so order holds.
- `Avatar`'s welcome payload awaits `MessageApi.topicsWarming()` before
  reading `MessageApi.topics().snapshot()`.
- A synchronous read (`descriptor`, `snapshot`, `isCommunicative`) that
  finds the dictionary cold throws `DocumentNotResidentError` — unreachable
  through `Scene.send`.

Access is through `MessageApi` and the message logic (`isCommunicative`,
`topicsWarming`); there is no `TopicCatalogueApi`.

## Auto-fallback for unknown topics

`MessageApi.topics().descriptor` walks three tiers in order:

1. **Cache hit** — return the authored entry verbatim.
2. **Family-inherited** — walk the dotted-path chain
   (`segments.slice(0, i).join('.')` for `i` from `length - 1` down
   to `1`) looking for the nearest authored ancestor. If found,
   inherit the family's description and synthesize the label as
   `<family-label> (<leaf-titlecased>)`.
3. **Derived default** — titlecased last segment as the label,
   `'(no description)'`, family = path prefix. ⚠ **This tier files a
   diagnostic.** Reaching it means a topic is emitted that nobody
   authored; the frame still renders, but the omission is reported
   instead of hidden. Fires once per key, not once per frame — a chatty
   topic would otherwise flood the diagnostics store.

The family-inheriting step scales dynamic-topic generators: an authored
family descriptor provides a useful label and description for every
runtime-composed leaf beneath it without per-leaf hand-authoring.
Authors override by writing a specific topic document; the cache hit at
step 1 wins.

The client runs the **same three-tier resolution** against its
cached snapshot — descriptor lookups don't round-trip.

## Wire push on session-establish

At session-establish, the avatar's welcome payload awaits the warm and
reads `MessageApi.topics().snapshot()` into its `topicCatalogue` field. The client
caches the array in a `Map<topic, TopicDescriptor>` on the Zustand
store; `getTopicDescriptor` consults that map.

No live updates in v1. Mid-session descriptor edits land on next
login.

## Cache invalidation

⭐ A pack sync that touches the `topic` kind calls `MessageApi.topics().invalidate()`
(the installer's per-kind go-live), which drops the dictionary; the next
scene re-warms it, queueing behind that warm like the first. No restart.

## `communicative`

The optional **`communicative: true`** flag marks a topic as a
*communication act* (say/whisper/shout/emote/chat — **not** dm /
narration / system). The dictionary builds a server-side `Set` of these
as it installs (no `TopicDescriptor` wire change) and exposes
`isCommunicative(topic)`, surfaced through `MessageApi.isCommunicative`.
The renown reception gate (`SensorMixin.onMessage`) consults it so only
genuine comm frames mint a being-heard signal — the first data-driven
capability hung on a topic. See [renown.md](./renown.md).

## ⭐ The six facets — attention, in data

The dotted tree expresses exactly **one** hierarchy. Everything that
cuts across it — *everything addressed to me*, *everything that
matters*, *what should interrupt* — used to be a client-side lookup
table keyed on ~90 topic strings, which drifted from the seeds every
time a topic was added. The knowledge lived in the wrong repository.

Five fields on every descriptor put those answers in the data:

| Facet | Values | What the client does with it |
|---|---|---|
| `address` | `direct` · `personal` · `ambient` · `broadcast` | Badging and notification. `direct` earns a push; `ambient` never does. |
| `actor` | `self` · `person` · `world` · `system` | Gutter colour and voice — replaces colour-by-family, which encoded the *emitter*. |
| `weight` | `consequence` · `activity` · `chatter` · `diagnostic` | Default filter levels. "Quiet mode" is `weight ≤ activity` — **one rule**, not sixty paths. |
| `audience` | `player` · `author` · `all` | Which surface it belongs to. |
| `durable` | boolean | Keep in scrollback and transcripts, or let it age out. |

**Both halves are needed.** Facets fix cross-cutting queries; only the
*tree* fixes subtree mutes ("everything about the air in here"), which
is a prefix operation. Neither replaces the other.

They ride the existing `TopicDescriptor` snapshot — no new channel, no
new endpoint. All five are **required** on the type: a facet that is
sometimes absent puts the fallback back in the client, which is the
defect they exist to remove.

### All three resolution tiers produce them

1. **Authored** — read from the document verbatim, following the
   `communicative` precedent.
2. **Family-inherited** — a leaf takes its ancestor's *attention shape*
   along with its prose. A child of `world.chat` is chatter for the same
   reason its parent is.
3. **Derived default** — the **conservative floor**: `ambient` /
   `system` / `diagnostic` / `all` / not-durable. An unknown topic must
   be **quiet, not loud**; the failure mode being designed against is a
   topic added without facets silently interrupting every player.

### ⚠ There is no family-prefix derivation at read time

The mechanical derivation (`system.*` → `system`/`all`, and so on) lives
**only** in `scripts/derive-topic-facets.ts`, which bakes its answers
into the documents. `Topic.readFacets` is authored-or-floor and
nothing else.

That is deliberate and was learned the hard way: the build first had a
derivation in the script *and* a second one in the catalogue, and the
two disagreed about `world.speech.say`'s `weight` within an hour of both
existing. A second taxonomy describing what the first already knows will
drift. **The topic document is the single source of truth.**

### The exceptions are the point

`derive-topic-facets.ts` derives all 89 seeds and carries **24 hand
exceptions**. A derivation needing *no* exceptions would prove the
facets carry nothing the path did not already have; the exceptions are
exactly where attention and emitter disagree.

The sharpest is the speech family splitting three ways on one facet —
`say` is `broadcast`, `whisper` is `personal`, `dm` is `direct` — which
is precisely what the tree structurally cannot say. A test pins that
split: if it ever collapses to one value, the facet has stopped earning
its place.

## Topics introduced by the social-cluster build

| Topic | Where it lives | Producer |
|---|---|---|
| `world.expression` | family root | (engine) |
| `world.expression.emote` | leaf | catalog + free-form emote frames (`SoulMixin`) |
| `world.speech.whisper` | leaf | `VocalMixin.whisper` |
| `world.speech.shout` | leaf | `VocalMixin.shout` |
| `world.chat.message` | leaf | chat posts (`ChannelCatalogue.postToChannel`) |
| `system.broadcast` | leaf | `broadcast` verb (`BroadcastController`) |
| `system.shell.chat` / `system.shell.contacts` / `system.shell.group` | leaves | chat / contacts / group verb feedback |

The emote leaf shares the modality `'emotive-esp'`; chat posts and
broadcast both ride `'verbal-esp'`. See
[emotes.md](./emotes.md), [chat.md](./chat.md), and
[messaging.md](./messaging.md) for the producer side.

## History

- 2026-11-09 — documents path identity: topics became `topic` documents
  at `content/topics/`; `TopicCatalogue`, the `Topic` Idea, its boot line
  and `pruneRetiredRoots` were deleted, and the vocabulary loads on the
  first scene behind an ordered dispatch queue. See git log for the
  `reqs/documents-path-identity` branch. (The "facets need a reseed"
  caveat went with the insert-only seeder: the installer is three-way,
  and a topic sync re-warms.)
