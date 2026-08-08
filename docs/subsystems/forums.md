# Forums

Forums are durable, addressable, multi-author boards riding the aether
implant — the persistent twin of [chat](./chat.md). Where a chat channel
is an ephemeral 200-frame ring, a forum board, every entry, and the vote
tally **persist** as Documents in their own collections; the board *is*
the archive. One board primitive (`Subject → Board → Thread → Post`)
carries a per-board **organizer** axis: `organizer: 'popularity'` (the
Reddit shape — cycle 1) and `organizer: 'argument'` (the **argument-map**
— a typed claim-graph read by a neutral structural lens, no ranking;
cycle 2). Both organizers persist into the same collections; the axis is
a field, never a schema migration.

The governing thesis: **popularity forums and structured-argument
deliberation are two organizers over one board primitive, not two
subsystems.** What separates them is how sibling entries are ordered and
how votes are read — not the bones (persistence, audience, access,
threading, command surface). This doc is the source of truth for the
area (both organizers); read it before editing. The argument organizer
has its own section below ([The argument organizer](#the-argument-organizer-cycle-2));
the cooperative slate's *Deliberation* section is its governance framing.

## The Subject layer — the linking spine

A `Subject` (`lib/forum/Subject.ts`, Document, `forum_subjects`) sits
*between* a surface and its `GroupRef` audience. It lifts the identity +
audience that chat formerly split across `Channel.owner` +
`Channel.groupRef` + name into **one** record, so a single subject can
light up *à la carte* across surfaces, all inheriting one `groupRef`.

Fields (persistent, public for the Hydrator):

- `title` — the addressing handle: flat-global + unique for venues,
  board-scoped (`board/thread`) for promoted topics.
- `owner` — the creator's playerId; the **mutation gate** (lifted from
  `Channel.owner`).
- `groupRef` — the audience binding (lifted from `Channel.groupRef`),
  with three shapes mirroring chat's three channel kinds:
  - **curated** — a managed Group minted alongside the subject;
    `groupRef = 'managed:<id>'`. The managed group carries
    owner/admin/member roles.
  - **bound** — an existing `guild:`/`mql:`/`contacts:` ref; the subject
    mints nothing, it just consumes the membership.
  - **open** — `groupRef` empty; audience is every player.
- `lifecycleClass: 'standing' | 'ephemeral'` — cycle 1 always
  `'standing'` (the ephemeral bill lifecycle is deferred).
- `state: 'active' | 'archived'` — `'active'` until the deferred archive
  cascade flips it.
- `manifestations: { surface, ref }[]` — the lit surfaces.
- `grain: 'venue' | 'topic'` + `parentSubject` + `boardScopedName` — the
  grain polymorphism (below).

### The four surfaces

A subject declares **any non-empty subset of four surfaces**, at most
one of each (`SubjectSurface`):

| Surface | Backing doc | Cycle 1 |
|---|---|---|
| `popularity-forum` | `Board`, `organizer: 'popularity'` | **shipped** (cycle 1) |
| `argument-forum` | `Board`, `organizer: 'argument'` | **shipped** (cycle 2) |
| `free-chat` | `Channel`, `procedure: 'free'` | **shipped** |
| `rules-chat` | `Channel`, `procedure: 'rules-of-order'` | reserved (parked) |

So *forum* = {popularity, argument} (the two `Board` organizers) and
*chat* = {free, rules} (the two `Channel` procedures). A subject may hold
both organizers and/or both chat procedures at once. Both forum surfaces
now ship (`popularity-forum` cycle 1, `argument-forum` cycle 2); `free-chat`
ships, `rules-chat` is parked.

### Subject grain — venue vs topic

A surface manifests at the subject's grain:

- **venue** (board-grain) — a standing `Board` / board-wide `Channel`,
  addressed by a flat-global `title`. Gossip, a guild board.
- **topic** (thread-grain) — a **promoted** `Thread` on a parent board,
  addressed by the board-scoped `parentSubject/boardScopedName` handle.
  Promotion mints a thread-subject whose `groupRef` *inherits the
  parent's* (`SubjectCatalogue.makeThreadSubject`).

Cycle 1 exercises both grains. Promoted thread-subjects are **standing**
(manually disbanded); the ephemeral lifecycle + archive cascade (bills)
and the governance trigger remain deferred to the Compact-governance build.

### SubjectCatalogue — the runtime view

`obj/SubjectCatalogue.ts` is a singleton `Idea` + `PostRegistrationMixin`
at `/obj/SubjectCatalogue`, named after its root entity like
`ChannelCatalogue` / `SoulCatalogue`. The `forum_subjects` Documents are
the source of truth; the catalogue warms them at `postRegister` via
`Subject.find({})` and serves resolution off in-memory indexes:

- `byId: Map<subjectId, Subject>` (the warm-gate) + `byTitle:
  Map<lowerTitle, Subject>` — case-insensitive title resolution.
- The `RESERVED_NAMES` set — a subject title must not collide with a
  subcommand word (lifted from `ChannelCatalogue`).
- `backedGroupIds: Set<string>` — the managed Group `_id`s minted to
  back curated subjects, so `group list` can filter them out (the chat
  `getBackingGroupIds` precedent).

`makeSubject(owner, title, opts)` mints the curated managed group by
default; `opts.groupRef` binds an existing ref (mints nothing);
`opts.open` leaves it empty. Audience membership is read **read-only**
through `GroupApi` (`isMember` / `parseRef`) — a subject is a *consumer*
of the group substrate, never a provider. `visibleSubjects(actor)` lifts
chat's `visibleChannels`: open subjects always shown, ref-backed shown to
members.

### Subscription storage — `SubjectSubscriberMixin`

Per-subject subscription state lives on the **owner** (Avatar) via
`SubjectSubscriberMixin` (`lib/forum/SubjectSubscriber.ts`), a typed
keyed store `_subjectSubscriptions: Record<subjectId, SubjectSubscription>`
persisted by reflection (the `ContactsMixin._contacts` /
`AliasMixin.aliases` structured-value precedent). One entry per
subscribed subject:

```ts
interface SubjectSubscription {
  followed: boolean;          // one toggle tunes all the subject's surfaces
  mutedSurfaces: SubjectSurface[];  // per-surface mute overlay
}
```

The merge/default policy and the group-audience checks live on
`SubjectCatalogue` (`getSubscription` / `setSubscription` / `follow` /
`mute`); the mixin is the raw keyed store it delegates to. This
**replaced an earlier per-subject `PropertiedMixin` property bag** (a
review refactor) — the subscription surface is expected to grow
(notification prefs, digests), so it earns a typed home + method contract
now.

`migrateLegacySubscription` ports a retrofitted channel's legacy
`{tunedIn, muted}` onto the per-subject store **lazily on first read**,
idempotently (`tunedIn → followed`, `muted → 'free-chat'` in
`mutedSurfaces`). Lazy-on-read avoids a boot-time write storm.

## The hierarchy — Board → Thread → Post

- **`Board`** (`lib/forum/Board.ts`, `forum_boards`) — a long-lived venue
  holding many Threads. Fields: `subject` ref (every board belongs to a
  Subject — the universal root), `organizer: 'popularity' | 'argument'`,
  `name`, `description`, and an `override` bag (designed-in but inert —
  the deferred per-board moderation layer).
- **`Entry`** (`lib/forum/Entry.ts`, `forum_entries`) — a node in a
  board's reply tree. A **Thread** is a root Entry (`parent: null`)
  carrying a `title` + `body`; a **Post** is a child Entry (`parent` =
  another entry's `_id`). The `relation` edge is **organizer-scoped**: a
  `'popularity'` board uses only `'reply'` (a strict reply-tree,
  **Reddit-style nested** — a `parent` edge to any entry, arbitrary
  depth); an `'argument'` board uses only the typed claim-graph edges
  `'supports' | 'objects-to' | 'responds-to'` (see [The argument
  organizer](#the-argument-organizer-cycle-2)).

Entry also carries `author` (the durable id link — **never** a frozen
name; the display name resolves FROM the id at read time so renames
reflect and the byline can become viewer-aware later), an optional
`subject` ref (set when the thread is promoted), `state: 'active' |
'locked'`, `editedAt`, and the denormalized vote aggregate `up`/`down`
(`getScore() = up − down`).

## Source of record — CRUD docs + an append-only event log

Every forum mutation **dual-writes**: it updates the current-state
Document *and* records an event. The CRUD docs stay the **queryable
truth** (not event-sourced); the log is the **durable audit/archive
twin**; the live feed is the in-process `EventApi` bus.

`ForumEvent` (`lib/forum/ForumEvent.ts`, `forum_events`) is the
append-only log — the `ChronicleEntry` precedent: a plain Document, the
row IS the event, **never mutated or deleted in normal operation** (the
tamper-evident substrate the argument organizer needs). Kinds:
`board-created`, `post-created`, `reply-created`, `vote-cast`,
`thread-promoted`, `thread-locked`, and the argument-organizer kinds
`argument-attached` (a typed claim), `entry-edited` (a body edit —
`data.priorBody` carries the lossless trail), `mature` (the decoupled
mature→vote handoff). Every row populates **all four**
dependency keys (`subject`/`board`/`thread`/`entry`) so the subscription
dependency index routes correctly even for events that nominally touch
one level (a `vote-cast` carries its entry's board + thread so a
board-list subscription re-resolves).

### Persist-then-fire

The dual-write is the private off-class `recordEvent` helper in
`ForumsLogic`:

1. `await row.save()` — the durable `forum_events` row lands. Silent: no
   change-stream, `Document.save()` fires nothing.
2. `EventApi.fire(new ForumEventFired(payload))` — the transient
   in-process twin. Persists nothing.

The two are **independent siblings** of the mutation — neither causes the
other (the write isn't event-sourced; the event has no tail). The order
is **persist-then-fire** so no listener observes a live event whose
durable row hasn't landed. This mirrors the existing `fireFieldChange →
EventApi.fire` pattern (MQL-sub) and the `ChronicleApi.record` silent-
append precedent. `ForumEventFired` is a `BusEvent`-shaped DTO
(`static KIND = 'forum.eventFired'`) carrying the same dependency-key
payload (the `FieldChangedEvent` precedent).

`recordEvent` is off-class because the mutation methods reach it without
an intra-singleton self-call — a self-call would resolve the caller to
the logic module, not the Api, and fail the `FromModule` gate.

## The document-change observer — `ForumSubscriptionRegistry`

Forum live-reads are a **forum-specific document-change observer**
(`obj/ForumSubscriptionRegistry.ts`, singleton `Idea`) — its own thing,
**not** MQL-subscription and **not** a Mongo tail. It listens on the
in-process `EventApi` bus (`EventApi.on(ForumEventFired.KIND)`), routes
through a dependency index, `setImmediate`-batches the dirty
subscriptions, and **re-resolves the current-state `Board`/`Entry`
Documents** for each subscription's scope → diff → delta. It does NOT
read `forum_events` (the durable twin, read only for history/audit).

**Independent of MQL-subscription.** It is *informed by*
`MqlSubscriptionRegistry`'s observer pattern (registry + dirty-batch +
diff) but shares **no code** — MQL-sub observes the Stuff world-tree by
re-resolving a query; this observes Documents by re-reading them.
`MqlSubscriptionRegistry` is **not touched**, so its no-regression
constraint is structurally trivial (two independent observers over
different data; the only overlap is the pattern itself). The persist-
then-fire invariant is what keeps the delta correct — the registry only
ever re-reads *committed* CRUD docs, so a delta can never reflect an
event whose row hasn't landed.

State + flow:

- `registry: Map<Interactive, Map<subscriptionId, ForumSubState>>` —
  subscriptions are **keyed by Interactive** (the delivery target).
  Each `ForumSubState` carries the scope + `lastResult: Map<id,
  ForumEntryRecord>` for diffing.
- Three scope kinds (`ForumSubscriptionScope`): `index` (the forum
  landing — the boards the viewer can see), `board` (a board's
  thread-list), `thread` (a thread's post-tree). `byBoard` / `byThread`
  dependency indexes + an `indexSubs` set route fired events to dirty
  subscriptions.
- A scope is **delivered to the Interactive but projected through the
  holder Avatar** — `viewerOf(interactive)` resolves the holder and
  requires it compose `Sensor`; `projectScope` re-reads the docs and runs
  them through `ForumsApi.listBoards` / `readBoard` / `readThread`, which
  apply the viewer's audience filter. Re-resolved live each flush, so a
  subscription always reflects the current perception/access view.
- `normalizeScope` canonicalizes a board scope's `id` (which the GUI may
  pass as a flat title **handle**) to the board `_id` so the index keys
  match the event payloads; `safeFind` swallows the `BSONError` Mongo
  throws on a non-ObjectId id (a handle reaching `getBoard`).
- `routeFire` marks all `indexSubs` dirty on any event (the board set is
  small) plus the `byBoard`/`byThread` matches; `markDirty` schedules one
  `setImmediate` drain; `drainDirty` → `reresolveAndEmit` re-projects and
  diffs (`diffRecords` — a generic pure keyed-set diff, NOT MQL's;
  add/replace/remove ops keyed by id), shipping only when changes exist.
- The listener is **refcounted** (`ensureListener` / `releaseListener`):
  the first subscription subscribes to `EventApi`, the last teardown
  unsubscribes.

The wire frames (`@saxonberg/types`):
`forum-subscription-result` (initial snapshot, `records:
ForumEntryRecord[]`), `forum-subscription-delta` (`changes: ForumChange[]`,
the add/replace/remove ops), `forum-subscription-error`. A
`ForumEntryRecord` carries `up`/`down`/`score`/`displayScore` (true score
+ the gated display value), `parent` (for nesting), `authorName`
(resolved on read), `createdAt`, etc.

Authors are resolved id → name in **one batched lookup** per projection
(`resolveAuthorNames`): the live avatar's `getPresentation()` first, else
the durable Avatar template's name. Empty ids (anonymous guests) render a
generic byline. The name is never stored on the entry.

> **`setImmediate`, not `ScheduleApi`.** The registry uses `setImmediate`
> for dirty-batching, matching `MqlSubscriptionRegistry`'s sibling-observer
> pattern. This deliberately follows precedent rather than the
> `ScheduleApi`-over-bare-timers rule — the two observers stay consistent.

## Voting + the anti-snowball gate

Every Entry (thread roots + posts) is votable. A vote is a **toggle
state per `(entry, voter)` ∈ {up, none, down}**, one row per pair in
`forum_votes` (`Vote`, `lib/forum/Vote.ts`), with a unique compound index
on `(entry, voter)` enforcing **one-account-one-vote**. The Entry carries
a **denormalized `up`/`down` aggregate** recomputed from the vote store
on each cast, so the popularity sorts + the live score delta read off the
Entry without scanning votes.

`ForumsLogic.castVote(actor, entry, direction)`:

1. **Own entry is locked** — a voter `=== entry.author` is refused; the
   author's auto-upvote row is fixed.
2. Toggle the `forum_votes` row: same direction → delete (→ none);
   different → flip; absent → insert.
3. Recompute `entry.up`/`entry.down` from the store, `save()`.
4. `recordEvent({ kind: 'vote-cast', ... data: { value, up, down } })` —
   the live delta half.

**Auto-upvote own** — a fresh thread/post is created `up = 1` and
`seedAuthorVote` writes the author's locked `up` row (Reddit-style).

**Anti-snowball is display-only.** `displayScoreFor(entry)` returns the
true net score once the entry has accumulated **K votes OR aged T
minutes** (`forums.antiSnowball.minVotes` / `.minMinutes` via `AppApi`
cached reads, defaults 5 / 30), else `null` — the client renders a
placeholder (`···` / `[···]` on the CLI). **The server ALWAYS ranks on
true scores** (`sortEntries` reads `getScore()`); the gate never touches
ordering.

Sorts (`sortEntries`, applied at both the board thread-list and the
comment-sibling levels): `new` (newest first), `top` (net desc), `hot`
(`sign(net)·log10(max(|net|,1)) + ageSeconds/45000`, constants deferred
tuning), `controversial` (high volume × balanced up/down). Votes are
**discovery-only** — never wired to standing/money; flat
one-account-one-vote; trust-weighting deferred.

## The argument organizer (cycle 2)

The **argument-map** is the `organizer: 'argument'` reading of the same
`Board`/`Entry` store — the polity's load-bearing **deliberation
surface**, where a proposal is reasoned through before a vote. It is
**not new storage**: a typed claim-graph interpretation + verb mode + a
computed read projection over the documents the popularity organizer
uses. The governing principle (from the cooperative slate): **load-bearing
deliberation must be organized by the argument's structure, not by any
user-signal ranking** — in a gamified polity any outcome-affecting ranking
decays to popularity/exploit, so the only ungameable organizer is the
logic of the argument itself. Nothing is ranked here.

### The typed claim-graph

An argument board's tree is rooted at a `parent: null` **spine** (the
proposal, as prose — `forum post <board> <thesis>`). Every other `Entry`
is a typed claim carried by its `relation` edge — the node's *role* is
**derived from the edge**, there is no separate node-type field:

- `'supports'` — a pro (argues *for* its parent),
- `'objects-to'` — a con (argues *against* its parent),
- `'responds-to'` — the **neutral** edge: a question / clarification that
  takes no side.

Recursion is the point — a claim whose parent is itself a claim is a
*rebuttal* (a pro/con of an argument). Depth carries nuance; the graph
stays a **strict tree** in v1 (`Entry.parent` is a single ref — the DAG
case, a canonical claim reused under many parents, is the deferred
claim-dedup problem). The vocabulary is **organizer-scoped** and enforced
at contribution time by the off-class `legalRelationsFor(organizer)`:
`reply()` rejects on an argument board, `attachClaim()` rejects on a
popularity board. Claims are **reputation-blind** — never vote-seeded;
`up`/`down` are never read under this organizer (no `castVote` —
`ForumsLogic.castVote` and the `forum vote` verb both refuse).

### Store / lens split — the neutral default lens

How the graph is *stored* must not bake in how it's *read*. The store is
the dumb relation tree (the chronicle / belief-store idiom); the reading
is a **computed lens** (`ForumsLogic.readArgumentLens` /
`readArgumentThread` → `ArgumentLensNode[]`). The one shared read surface
is the **neutral default lens**: spine first, then each node's children
**grouped by valence** (Supporting → Objections → Questions),
chronological within a group — **never by score**, computed from pure
relations with no display-order or score field welded onto the `Entry`.
Because the only shared surface is structural, there is no shared target
to game (per-viewer lenses would be safe; v1 ships only the default lens
+ the highlight below).

**Open objection — the one dual-use metric.** A node is an *open
objection* when it is an `objects-to` with **no answering child** — a
literal hole in the argument, computed by `buildArgumentLens`. It is both
the best reading-triage cue and the convergence signal; it is unfarmable
(the only way to clear it is to *answer* it, which improves the map).
Answering = attaching any child; the flag clears live (the registry
re-resolves and `recordsEqual` fires a `replace`).

**Delegated attention — the circle highlight.** A per-viewer,
**non-reordering** overlay: a node is flagged `inCircle` when its author
is in the viewer's `contacts` circle (resolved read-only via `GroupApi`,
batched once per projection in `ForumSubscriptionRegistry.resolveCircle`).
You delegate *attention, not votes* — a captured trust-set can only
mis-route what you read, never what binds, and every node stays fully
present. Two viewers see different highlights over the **identical**
structural projection.

### Live reads — the organizer-aware projection

Argument boards ride the **same** `ForumSubscriptionRegistry` — the
single organizer-aware branch is in `projectScope`: when the resolved
board's `getOrganizer() === 'argument'` it routes through
`readArgumentLens`/`readArgumentThread` and `projectArgumentNodes`
instead of the popularity `readBoard`/`readThread`. The scope kinds, the
dependency index, the dirty-batch, and `diffRecords` are reused verbatim;
`MqlSubscriptionRegistry` is untouched. `projectArgumentNodes` zeroes
`up`/`down`/`score` and nulls `displayScore` (reputation-blind) and
stamps the argument-only `ForumEntryRecord` fields (`organizer`,
`relation`, `openObjection`, `inCircle`, `editedAt`) — all optional, so a
popularity projection is byte-identical when they're absent. `recordsEqual`
compares them so the open-objection badge, the edited marker, and the
highlight update live.

### Editing — edit-in-place with a lossless trail

Claims refine in place: `ForumsLogic.editBody` (author or board-subject
owner) re-MMLs the body and stamps `editedAt`, **and** records an
`'entry-edited'` event whose `data.priorBody` captures the prior text —
the read stays clean (one current claim) while the append-only log keeps
the lossless history (the grounded source the deferred dedup/summarization
LLM layer reads; the log is never compacted). The rendered claim carries
an **"edited" marker**; history surfaces through the deferred diff-lens,
not inlined. Body-only — re-parent / merge / split / delete are deferred
with claim dedup. **No edit notifications** (there is no notification
inbox yet and `follow` is passive); edits surface via the live delta + the
marker.

### The mature → vote seam

`forum mature <board>` (owner-gated, argument-only) calls
`ForumsLogic.matureArgument`, which records a `'mature'` event carrying
the board/subject keys (and the spine as `thread`/`entry`) and **binds
nothing** — the decoupled handoff the deferred vote / measure / docket
layer will consume. v1 emits the event into no consumer.

### The verbs + the client mode

Contribution reuses `reply` as a verb *mode*: `forum reply <node>
--pro|--con|--rebut` → `supports`/`objects-to`/`responds-to`
(`ForumController` maps the flags via `valence()`, requiring exactly one
on an argument board and rejecting them on a popularity board); `forum
edit <node>`; `forum mature <board>`. The React client renders an
**organizer-gated argument mode** in `ForumView` (additive — the
popularity thread-list/post-tree path is untouched): a prominent spine,
the recursive valence-grouped tree (`ArgumentNode`) with +/−/? markers,
the open-objection badge, the circle highlight, the edited marker, a
three-way contribution affordance, and the owner "Mature" action. No vote
controls, no scores, no popularity sorts. Every affordance builds a real
`forum …` command string through the shared command-click/preview path;
contributions render live via the subscription. The board index badges
argument forums (`⚖ argument forum`).

### Deferred (the scale problems)

The v1 slice is the small-and-safe claim-tree. The *scale* work is
deferred (see the argument-map slate):
**claim dedup / canonicalization** (assisted curation — the make-or-break
problem, and the DAG case), **integrity-grade map-summarization**
(grounded, drillable), **automated convergence-detection** (+ the
anti-railroad floor), **proposal version-control** + map re-anchoring (and
response version-anchoring with it), node refactoring, the **vote
consumer** of the `mature` event, edit **notifications** + an inbox, and
the rich plural-lens **explorer** (guided tours, question-lenses, diffs,
spatial renders).

## The aether capability — a born-with `ForumsUpdate`

Forums ride the existing aether implant as a **hosted update**, exactly
like DMs — **no separate physical implant**.
`ForumsUpdate` (`lib/forum/ForumsUpdate.ts`) is
`ForumsMixin(AetherHostedMixin(Idea))`, the sibling of `CommsUpdate`.
`ForumsMixin` (`lib/forum/Forums.ts`) is the capability gate + verb
contributor + operator resolver: it confers read + post, contributes
`commandContributions.self = ['social/forum.yaml']`, resolves its
operator via `getHost()`, and gates on `MixinApi.isAether(operator)`
(throwing `InactiveCapabilityError`) — the `CommsMixin.tell` pattern,
including the one documented `AetherHosted` cast naming the orthogonal
co-composition.

It is **born-with**: `Avatar.installDefaultLoadout` clones a
`ForumsUpdate` and `hostUpdate`s it onto the Avatar's aether, alongside
the comms + travel-credential updates. The `forum` verbs surface on the
host's recency stack via the hosted-update self-seeding
(`CommandApi.collectHostedUpdateDefs` + `applyHostedUpdateDelta`); the
controller reaches the holder's update via the `commandSource` that
afforded the verb, else the MQL `reachable` pool filtered on
`MixinApi.isForums` (the `DmController` fallback).

## The command body side-channel

The GUI builds **real command strings** (verb + selectors) — every action
stays scriptable / aliasable / replayable. Input that can't sit on a
command line (a multi-line post **body**) rides an **optional structured
payload alongside the string** (HTTP method+path vs body): the existing
`{ type: 'command', payload: { text, fields? } }` inbound
(`backend/inbound/command.ts` reads `fields` into
`ExecuteCommandOpts.bodyFields`).

`text` is **always present and parsed normally**. After the parser binds
the model, `CommandGiver.executeCommand` calls
`CommandLogic.overlayBodyFields(model, fields, command)` (before
`resolveModel`), then runs the **identical** resolve → validate →
controller → envelope chain. The structural narrowness: `overlayBodyFields`
overlays `fields` **only onto the command's `payload:`-block fields or a
designated body arg** — a greedy `string` positional. Options (flags) and
object/MQL selectors are **unreachable by design**, so the side-channel
can never fill a selector or toggle a flag. A value that fails coercion is
left unset → downstream validation rejects it.

There is **no string-less dispatch path** exposed to clients
(`assembleFromStructured`'s string-less mode stays an internal helper).
Reuses the existing `payload:` block + `struct` type (the `write.yaml`
precedent) — **no command-spec schema change**.

### Three routes to one MML body

A post body is **MML**, parsed with the **article dialect** and the
`inert` tag policy (`longForm: true, tags: 'inert'`) — headings, nested
lists and pipe tables, the same markup a wiki article gets, because a
post is long-form authored prose composed through the same prompt. It
parsed the *chat* dialect until 2026-08-04, so a post could not carry a
heading.

> ⚠ `inert`, not the wiki's `all`. The wiki can admit every tag because
> its pipeline resolves components against a charset-restricted
> namespace and gates spoilers; a post is stored as MML and rendered
> directly, so it admits **presentation only** — no `<link>` or
> `<mention>` (the client turns those into clickables that ISSUE
> COMMANDS) and none of the identity tags the composer emits on the
> server's authority. See [wiki.md](./wiki.md) § the tag policy.

Conversion happens on the way IN, so existing posts are unaffected.

(markdown → `Mml.markdownToMml` server-side →
stored MML → `MmlRenderer` displays). `ForumController.resolveBody`
reaches one body field three ways:

1. **greedy markdown on the command string** (CLI / scripting) — overlaid
   into `model.body` by the parser.
2. the **`fields` side-channel** (GUI) — overlaid by `overlayBodyFields`.
   Lean `fields` wins when both inline-greedy and the side-channel supply
   the body.
3. an interactive **`compose` prompt** (`PromptApi.compose`, a new shared
   PromptApi/client kind, forums first then CMS/wiki) — a multiline
   `<textarea>` (markdown, ⌘/Ctrl+Enter submits) with an "open in editor"
   escalation hook stubbed for the deferred CMS rich editor.

## The verb model — `subject` + `on`, `make` as sugar

Identity + audience own a `subject` verb; surfaces light up separately.

- `subject make <name>` — mint a subject. Default mints a managed group
  (curated); `--group <ref>` binds an existing ref; `--open` leaves the
  audience open. `subject list` — visible subjects + their lit surfaces.
  (`obj/command/social/SubjectController.ts` + `mud/cmd/social/subject.yaml`.)
- `forum on <subject>` / `chat on <subject>` — attach a surface to an
  owned subject (gated by `Subject.owner`). `--argument` (forum) /
  `--rules` (chat) select the deferred organizer/procedure (flag ships,
  behavior doesn't).
- `forum make` / `chat make` survive as **sugar** — subject-make +
  light-up in one step, default popularity/free. `make` errors on a taken
  name; `on` attaches to existing and is denied to a non-owner.

`forum` (`ForumController` + `forum.yaml`) has subcommand fallthrough (the
`chat.yaml` pattern — bare `forum <board>` reads): `list / make / on /
post / reply / read / vote / promote / follow`. A **board-subject's title
is a flat-global handle** (case-insensitive, unique; `RESERVED_NAMES`
carries over); a **thread-subject's handle is board-scoped**
(`board/thread`). A plain thread is **promoted** to a thread-subject with
its own chat via `forum <board> promote <thread> <name>` →
`chat on <thread>` (the `promoteAdHocToManaged` precedent;
`ForumsLogic.promoteThread` mints the thread-subject + stamps
`Entry.subject`).

### Chat retrofit (Wave 0)

Introducing Subjects refactors chat: `Channel` drops `owner` + `groupRef`
(they move up to `Subject`) and gains a `subject` ref + a `procedure:
'free' | 'rules-of-order'` field. The three channel kinds map:
player-created → a **curated** subject, open-join-standalone → an
**open** subject, ad-hoc → runtime-only (no subject, unchanged).
`ChannelCatalogue` reads `groupRef` *via the Channel's Subject* and
delegates discovery + subscription to `SubjectCatalogue`. The
`Channel.kind` strings are **kept and mapped** at read (no enum rename,
no data migration of existing docs). Existing chat behavior must not
regress — both player-created and open channels keep working, now exposed
as curated/open subjects.

## The Apis & module map

| Element | Lives in | Role |
|---|---|---|
| `Subject` | `lib/forum/Subject.ts` | Linking-spine Document (`forum_subjects`) |
| `Board` / `Entry` / `Vote` | `lib/forum/` | Forum Documents (`forum_boards`/`_entries`/`_votes`) |
| `ForumEvent` + `ForumEventFired` | `lib/forum/ForumEvent.ts` | Append-only log row (`forum_events`) + its transient bus twin |
| `ForumsMixin` / `ForumsUpdate` | `lib/forum/Forums.ts` / `ForumsUpdate.ts` | The capability mixin + the born-with hosted aether update |
| `SubjectSubscriberMixin` | `lib/forum/SubjectSubscriber.ts` | Per-Avatar typed subscription store |
| `SubjectCatalogue` | `obj/SubjectCatalogue.ts` | Subject runtime view + subscription store + group bookkeeping |
| `ForumSubscriptionRegistry` | `obj/ForumSubscriptionRegistry.ts` | The document-change observer |
| `ForumsApi` / `ForumsLogic` | `api/forums.ts` / `obj/api/ForumsLogic.ts` | Board/Entry CRUD + voting + promotion + the folded-in subscription surface |
| `SubjectApi` / `SubjectLogic` | `api/subject.ts` / `obj/api/SubjectLogic.ts` | Subject identity/audience facade |
| `ForumController` / `SubjectController` | `obj/command/social/` | The `forum` / `subject` verbs |

`ForumsApi` is the thin gated forwarding facade over `ForumsLogic` (the
HMR-able logic singleton at `/obj/api/forums`); both follow the
`ChatApi`/`ChatLogic` split. The **live subscription surface**
(`handleSubscribe` / `handleUnsubscribe` / `cancelAllForInteractive`) is
**folded into `ForumsApi`** (one forum-facing Api), forwarding to the
separate `ForumSubscriptionRegistry` runtime singleton — a review refactor
(it was once its own `ForumSubscriptionApi`). The registry's gated
methods carry `@CallSecurity(FromModule('/api/forums#ForumsApi'))`, so
the facade is the only legitimate caller. The `forum-subscribe` /
`forum-unsubscribe` inbound handlers and the disconnect teardown call
through it.

The new collections registered with `PersistenceManager`:
`forum_subjects`, `forum_boards`, `forum_entries`, `forum_votes`,
`forum_events`.

## Client channels

The forum is a **primary view inside the cockpit**, not a
`connectionPhase` — adding a phase would hide the terminal/command-bar (a
leave-the-game context switch). Instead a `mainView: 'terminal' | 'forum'`
slice renders `Terminal` or `ForumView` in the LeftColumn content slot
while **Frame + CommandBar persist**, the **right column is
view-sensitive** (terminal → `InspectionPane`; forum →
`ForumChatSidecar`), and live scene events still surface as a peek/toast.
Navigation is verb-driven (`forum` / `forum <board>` flips `mainView` +
the nav target) and click-driven; the player stays in
`connectionPhase: in-world`.

- **`ForumView.tsx`** browses board → thread → post-tree, rendered **live**
  by three subscriptions (`index` / `board` / `thread`, via
  `forumActions.subscribeForumScope`). Sorting is client-side over the
  live record set (the server always ships true scores; the display gate
  rides `displayScore`). The post-tree nests recursively (`CommentNode`),
  siblings sorted per level.
- **Client-owns-UI-state.** Mode, filters, sort selection, and compose
  drafts are client React state — they never ride the bus. Only data
  (the subscription records) and actions (command strings) cross the wire.
- **The GUI builds REAL command strings.** Votes send `forum vote <id>
  up|down`; post/reply send `forum post <board>` / `forum reply <id>`
  with the multi-line body on the **`fields` side-channel**
  (`websocketClient.sendCommand(verb, { body })`). Every affordance routes
  through the shared `onCommandClick` / `onCommandPreview` handlers
  (preview in the command bar on hover, send on click) exactly like every
  other clickable — a vote click previews `forum vote …`, never a raw
  websocket frame. (Hover preview keys on `mousemove`, not `mouseenter`,
  so a live reorder under a still cursor can't re-stick a command.)
- **`ForumChatSidecar.tsx`** is the contextual chat rail: a client-side
  filtered slice of the message feed (frames whose `channelName` matches
  the board) + a **scoped-input "Talk here"** mode that puts the command
  bar into a `chat <handle>` prefix. v1 is **board-scoped only**; the
  promoted-thread nested chat (the subject-path stack) is deferred.
- `forumActions.ts` is the IO boundary: it wires the
  `forum-subscription-result` / `-delta` / `-error` envelopes into store
  actions (`applyForumResult` / `applyForumDelta` / `clearForumSubscription`)
  and offers the outbound ops; the store slice (`forumRecords` keyed by
  `subscriptionId`, `forumNav`) stays pure state.

## Open questions / deferred

- **The latent "watch a collection for changes" abstraction (deferred,
  on purpose).** `ForumSubscriptionRegistry` is instance #1 of a possible
  generic "observe a Mongo collection for changes" layer, and likely not
  the only future watcher — but the generic version is **deliberately not
  built now** (no second consumer yet to shape the seam; speculative
  generality risks the wrong abstraction). The forum instance is kept
  clean (its registry, the re-resolve/diff, a pure keyed-set diff) so it
  *can* seed the abstraction later. It is **not** a generalized MQL engine
  and **not** a shared forums/wiki/CMS engine (CMS/wiki are
  request-response authoring, not live subscribers). When a second real
  watcher appears, extract the generic layer from the two concretes.
- **The argument organizer / argument-forum** — the typed claim-graph,
  `supports`/`objects-to`/`responds-to` edges, the neutral default lens,
  open-objection, the circle highlight, the mature seam, and the client
  argument mode — **shipped (cycle 2)**; see [The argument
  organizer](#the-argument-organizer-cycle-2) above. Its *scale* problems
  (claim dedup, summarization, convergence, version-control, the vote
  consumer, the plural-lens explorer) remain deferred — see the
  argument-map slate.
- **Subscription query/pagination model.** Cycle 1 scopes a subscription
  to a single board's thread-list or a single thread's post-tree;
  deep-subtree paging is a structured query and the boundary between a
  live-subscribed window and a paged window needs a concrete cut when deep
  trees / large claim-graphs arrive.
- **The promoted-thread nested chat (GUI).** `ForumChatSidecar` is
  board-scoped only in v1 — the subject-path stack (the thread's
  thread-scoped chat alongside the still-reachable board chat) wants the
  thread-subject's board-scoped handle, which the client doesn't carry
  yet. Deferred.
- **Migration of existing chat subscription state.** Wave 0 rewrites
  `chat.subscription.<channelId>` keys to the per-subject store
  lazy-on-read (idempotent) — safer than a one-shot-at-warm write storm.
- **Ephemeral-subject lifecycle + archive cascade** (bills) and the
  **governance trigger** — deferred to the Compact-governance build (the substrate
  exposes mint / light-up / archive; *who calls it* is out of scope).
- **The rules-of-order chat procedure** (`procedure: 'rules-of-order'`) +
  chat `logged` retention / `chat_log` — declarable in the taxonomy, parked.
- **Measure / docket surfaces** — the third and fourth conceptual surfaces
  a subject could light up (polling / scheduling) are out of scope for
  this cycle.
- **Reactions on entries** — reusing the act-scoped
  [reaction](./reactions.md) layer on persisted entries is an open bridge
  (act-scoped/ephemeral vs entity-scoped/persisted); deferred.
- **Trust-weighted votes; votes → standing/money** — the cooperative
  slate's feedback economy; v1 votes drive discovery only.

## Related

- [chat.md](./chat.md) — the structural precedent (Channel Document +
  Catalogue + Api/Logic split + `groupRef`); the chat retrofit moves
  `owner`/`groupRef` up to `Subject`.
- [grouping.md](./grouping.md) — the `GroupApi` facade every subject binds
  read-only for audience/roles.
- [comms.md](./comms.md) + [augmentation.md](./augmentation.md) — the
  aether host ⊕ hosted-update relation `ForumsUpdate` rides (the
  `CommsUpdate` precedent; the MQL `reachable` seed's host-descent).
- [chronicle.md](./chronicle.md) — the append-only ledger precedent (dumb
  store / smart consumers, silent append) the `forum_events` log follows.
- [mql-subscription.md](./mql-subscription.md) — the *sibling observer*
  whose pattern (registry + dirty-batch + diff) informs
  `ForumSubscriptionRegistry`; the two share no code and MQL-sub is
  untouched.
- [command-routing.md](./command-routing.md) /
  [command-spec.md](./command-spec.md) — the `subcommands:` + `args:` +
  `fallthrough: true` shape; affordance attribution via `commandSource`.
- [response-envelope.md](./response-envelope.md) — controllers return
  `void`; outcomes ride the dispatch envelope.
- [prompt.md](./prompt.md) — `PromptApi.compose`, the new shared
  multiline-body prompt kind.
- [persistence.md](./persistence.md) — the `Document` track the forum
  collections ride.

## History

The forums cycle-1 build landed on `feature/forums-cycle1-build` (later
merged into `feature/forums-build`), sequenced as five
independently-testable waves: **Wave 0** (Subject layer + chat retrofit,
the foundational `feat(forums): Wave 0` commit `31b8961a`), then the forum
board substrate, the `forum_events` log + voting, the document-change
subscription + body side-channel, and the React GUI. Three review
refactors reshaped the surface mid-cycle:

- `ForumSubscriptionApi` was **folded into `ForumsApi`** (one forum-facing
  Api forwarding to the registry; commit `e24fe31d`).
- Per-subject subscription state **moved from a `PropertiedMixin` property
  bag onto `SubjectSubscriberMixin`** (a typed keyed store; commit
  `aff67854`).
- The forum subscription **teardown moved onto `Interactive`** (the
  Interactive owns its own disconnect teardown; commit `2b75d931`).

The **argument organizer (cycle 2)** landed on `feature/argument-map-build`
as four independently-testable waves over the cycle-1 substrate: relation/
organizer widening + contribution verbs (the `'structure'` → `'argument'`
rename, `EntryRelation` widened, `attachClaim`/`editBody`); the neutral
default lens + open-objection + circle highlight + the organizer-aware
`projectScope` branch; the decoupled `mature` seam; and the client argument
rendering mode in `ForumView`. The single behavioral change to the cycle-1
surface was the organizer value rename (no data migration — no board ever
persisted `'structure'`); everything else is additive. Its requirements +
plan retired at the cycle-2 sweep.

The forums-slate (forums-slate.md) is
retained for the remaining deferred design space (the ephemeral lifecycle,
the procedure modes, the latent collection-watch abstraction); the
argument organizer's scale tail lives in the
argument-map slate.
