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
| `dm` / `tell` | `address: direct` |
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
| `sense` | `ambient` `weather` `survey` `reading` | The world reaching you; `survey` answers your own looking |
| `self` | `body` `standing` `holding` `group` | Your own person |
| `publication` | `press` `wiki` `forum` | Authored durable content |
| `shell` | `result` `error` `prompt` `config` `control` `diagnostic` | The client↔server relationship |
| `session` | `link` `identity` `notice` `presence` | The connection itself |

### ⚠ The vocabulary is open; the roots are closed

Content packs **may** add topics — they install `domain` rows exactly as
core seeds do, and `TopicCatalogue` reads the collection, not a bundled
file list. Nothing special is needed.

**But a pack may add leaves only, never a root.** If a pack could mint a
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
| `lib/messaging/Topic.ts` | The `Topic extends Idea` leaf class with persistent fields `topic` / `family` / `label` / `description` and `TEMPLATE_PATH_PREFIX = '/obj/Topic/'`. |
| `obj/TopicCatalogue.ts` | The singleton Idea (`/obj/TopicCatalogue`) owning the runtime descriptor cache + accessor surface. Sibling to `obj/EventRegistry.ts` per the singleton-in-`obj/` convention. |
| `seeds/obj/Topic/<dotted-path>.yaml` | One file per topic leaf or family. Flat path strings, no nested directories. |
| `seeds/obj/TopicCatalogue.yaml` | Singleton seed (`{ class: /obj/TopicCatalogue, data: {} }`). |
| `bootstrap.ts` | Manifest entry for the `/obj/TopicCatalogue` singleton. **No per-Topic pre-clone** — the catalogue loads its own descriptors. |
| `@saxonberg/types` `TopicDescriptor` | Wire-safe shape: `{ topic, family, label, description }` + the six facets. `TopicRoot` / `TOPIC_ROOTS` live here too. Shared between the server snapshot and the client cache. |

## No code-side constants mirror

There is no `TOPICS` const in code. The earlier nested `TOPICS` tree
on `MessageApi` has been retired — topics are dotted-path string
literals at call sites (e.g. `.topic('speech.vocal')`), with
`Topic` Ideas + `TopicCatalogue` as the runtime catalogue. Three
deliberate non-things:

- No `Topics` enum exported from `@saxonberg/types`.
- No mirror file under `lib/topics/` (and no such subsystem dir —
  topics are a messaging concern, hence `lib/messaging/`).
- Topic strings cross the wire raw; the catalogue carries everything
  beyond the string itself (label, description, family).

The catalogue and the const are independent: descriptors are content
in the DB; `TOPICS` is a code-side autocomplete convenience.

## The Topic Idea

`Topic` extends `Idea` with four public string fields and per-field
invariants on the setters (empty strings rejected for `topic`,
`label`, `description`; `family` accepts `''` for root topics). The
`TEMPLATE_PATH_PREFIX` constant is `/obj/Topic/`. Authors
edit topic descriptors through the existing workspace shell —
there's no `describe-topic` verb.

`family` is encoded as a stored string field rather than derived
from the dotted path, so authoring can override the inheritance
chain if a future taxonomy refactor wants it. The catalogue's
fallback resolver uses the dotted-path structure directly.

## The TopicCatalogue singleton

`TopicCatalogue extends PostRegistrationMixin(Idea)`. Three instance
methods compose its surface:

- `getDescriptor(topic: string): TopicDescriptor` — three-tier
  resolution; **always returns a populated descriptor**.
- `getSnapshot(): TopicDescriptor[]` — flat array of authored
  entries, used by the wire push.
- `invalidateCache(): void` — drops the cached map. Future admin
  verb hook for "I just edited a Topic seed, re-read it without
  restarting"; currently unused at runtime (cache builds once at
  boot).

Cache state lives as a `Map<string, TopicDescriptor> | null` private
instance field, warmed by `postRegister`'s call to
`Template.findDescendants('/obj/Topic/')`. Resolution
dispatches through the standard call-security gate via
`StuffApi.findByTemplatePath('/obj/TopicCatalogue')` — there's no
`TopicCatalogueApi` indirection. Per
[[feedback-no-new-apis-default]], the singleton's instance methods
are the access surface.

**No pre-cloning of Topic templates.** Topic templates are pure
data (`topic` / `family` / `label` / `description`) with no runtime
behavior, so the catalogue reads `data.*` directly off each
`Template` doc — no `Topic` Stuff instances are ever cloned at
boot. The bootstrap manifest carries `/obj/TopicCatalogue`
but **not** a `templatePathPrefix: '/obj/Topic/'` entry.
Same pattern as species clades / materials / biomes (see the
preamble in `mud/bootstrap.ts` for the precedent).

## Auto-fallback for unknown topics

`getDescriptor` walks three tiers in order:

1. **Cache hit** — return the authored entry verbatim.
2. **Family-inherited** — walk the dotted-path chain
   (`segments.slice(0, i).join('.')` for `i` from `length - 1` down
   to `1`) looking for the nearest authored ancestor. If found,
   inherit the family's description and synthesize the label as
   `<family-label> (<leaf-titlecased>)`.
3. **Derived default** — titlecased last segment as the label,
   `'(no description)'`, family = path prefix. ⚠ **This tier now files a
   diagnostic.** Reaching it means a topic is emitted that nobody
   authored; the frame still renders, but the omission is reported
   instead of hidden. Fires once per key, not once per frame — a chatty
   topic would otherwise flood the diagnostics store.

The family-inheriting step scales dynamic-topic generators. For
example, `MudlogApi` composes `system.log.<category>.<level>` at
runtime (`system.log.command.info`, `system.log.command.warn`); the
authored `system.log.command` family descriptor provides a useful
label and description for both leaves without per-leaf hand-seeding.
Authors override by writing a specific Topic seed; the cache hit at
step 1 wins.

The client runs the **same three-tier resolution** against its
cached snapshot — descriptor lookups don't round-trip.

## Wire push on session-establish

At session-establish, `Avatar.enter` reads
`(await findByTemplatePath('/obj/TopicCatalogue')).getSnapshot()`
into the welcome scene payload's `topicCatalogue` field. The client
caches the array in a `Map<topic, TopicDescriptor>` on the Zustand
store; `getTopicDescriptor` consults that map.

No live updates in v1. Mid-session descriptor edits land on next
login. If real demand for mid-session updates appears, the catalogue
can be turned into an MQL-subscribed live query — but the snapshot
shape is forward-compatible with that move.

## Cache invalidation

The cache builds once at boot from mongo and stays put for the
process lifetime. Descriptor edits written to mongo during the
process are picked up on next boot (or via a future admin verb
that calls `invalidateCache` + `postRegister` to re-read). Since
the catalogue no longer holds runtime `Topic` Stuff instances,
there are no `Events.StuffCreated/Destructed` subscriptions to
manage.

The catalogue itself is a system singleton; `canDestruct` refuses
unconditionally (mirrors `EventRegistry`).

## Contrast with EventRegistry

Both are singleton Ideas in `obj/` that own a per-X data shape:

| Aspect | EventRegistry | TopicCatalogue |
|---|---|---|
| Content shape | Transient per-event policy closures | Persistent per-topic prose (label, description, family) |
| Source of truth | Code-side `Events` enum + `defaultPolicyFor` table | `Topic` Ideas under `/obj/Topic/` |
| Code-side vocabulary | `Events` enum is the vocabulary | no code-side const — topics are string literals at call sites; descriptors are content |
| Auto-resolve behavior | `EventApi.on/emit` auto-registers unknown events with the default `emittableBy()` policy | `getDescriptor` auto-falls-back via family inheritance or derived default |
| Persistence | Empty seed; runtime state is closures | Empty seed; runtime cache reads `Topic` template docs from mongo at boot |

## Seed YAML structure

Every topic is one YAML file at
`seeds/lib/messaging/Topic/<full-dotted-path>.yaml`. Flat path
strings — no nested directories under the `Topic/` prefix, no
`FolderZone` admin scaffolding at v1. The hierarchical meaning is
encoded in the `topic` / `family` fields.

```yaml
class: /lib/messaging/Topic
hydratorClass: /obj/persistence/PersistentHydrator
data:
  topic: world.speech.say
  family: world.speech
  label: Say
  description: A character speaks aloud to everyone in the room.
  communicative: true
```

System-owned at v1. Revisit FolderZone admin scoping when a doc team
emerges that needs scoped editing rights.

The optional **`communicative: true`** flag marks a topic as a
*communication act* (say/whisper/shout/emote/chat — **not** dm /
narration / system). `TopicCatalogue` builds a server-side `Set` of these
during its template scan (no `TopicDescriptor` wire change) and exposes
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

1. **Authored** — read from `data:` verbatim, following the
   `communicative` precedent. `Topic` gains no fields; Topic templates
   are pure data rows that are never cloned.
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
into the seeds. `TopicCatalogue.readFacets` is authored-or-floor and
nothing else.

That is deliberate and was learned the hard way: the build first had a
derivation in the script *and* a second one in the catalogue, and the
two disagreed about `world.speech.say`'s `weight` within an hour of both
existing. A second taxonomy describing what the first already knows will
drift. **The seed file is the single source of truth.**

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

### ⚠ Facets need a reseed, not a migration

The seeder is **insert-only**. Editing a seeded topic's `data:` does
nothing on a database that has already booted — drop the `/obj/Topic/`
rows and restart. On the wiping demo instance this is simply the
standing loop for reference data.

## Boot sequence

1. `SeederManager` inserts every YAML into the `domain` collection
   (including the per-topic seeds — those just sit in mongo as
   template docs, no runtime presence).
2. `BootstrapManager` clones `/obj/TopicCatalogue` (and nothing
   else in the messaging substrate).
3. `TopicCatalogue.postRegister` reads every Topic template via
   `Template.findDescendants('/obj/Topic/')` and warms
   the descriptor cache.
4. Welcome-scene payload composition reads `getSnapshot()` and
   ships it to the client.

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
