# belief — per-viewer identity memory (recognition & identification)

A **per-viewer memory of identity**: what a given viewer knows about the
things around it, and how the world's display names bend around that
knowledge. The same Bob renders as "Bob" to a friend, "a hooded figure"
to someone he's hiding from, "a tall stranger" to someone who's never met
him; the same vial reads "a blue potion" until identified, then "a potion
of healing."

Two **composable axes** of per-viewer knowledge, distinct but
co-applying, sharing one store, one persistence pattern, one masking
mechanism, and one compose seam:

- **Recognition** — *instance continuity*: "have I met this specific
  individual, and do I know who they are?" Keyed per individual.
- **Identification** — *type knowledge*: "do I know what kind of thing
  this is?" Keyed per class.

They dissociate (recognized-but-unidentified = "the stranger you keep
seeing"; identified-but-not-recognized = "a city guard") and compose
("Bob, a city guard"). A target gets, potentially, *both* lookups, woven
into one name.

## Three concerns that meet only at the naming step

- **Perception** (`docs/subsystems/perception.md`, unchanged) — *can V
  sense T at all?* Light, concealment, modality. Owns the visibility
  gate. This subsystem *calls into* it and otherwise leaves it alone.
- **Recognition / identification** (here) — *given V can sense T, what
  does V know about its identity?*
- **Self-presentation** (`Stuff.getPresentation()`) — *what does T call
  itself, viewer-blind?* The shared baseline the viewer-aware step builds
  on.

## The spine — `BeliefStoreMixin` (`lib/belief/BeliefStore.ts`)

A dumb, realm-namespaced keyed bag on `Character` (so every `Avatar` and
NPC carries it). Pure CRUD; all per-realm intelligence lives in consumers.

- Storage: `Map<string, BeliefRecord>` keyed `` `${realm}:${referent}` ``.
- Realms are string conventions, not a registry: `RECOGNITION`,
  `IDENTIFICATION`, `REGARD`, `DISCOVERY` exported consts.
- **Keyed on `referent.getTemplatePath()`**, never `stuffId` — `stuffId`
  is reboot-ephemeral and would imply the viewer "knows which runtime
  Stuff"; `templatePath` is durable and the engine always has it. The
  instance/type split falls out for free: a unique `templatePath`
  (avatars, singleton NPCs) is a recognition referent; a shared one
  (generic clones) is a type referent.
- Surface: `know(realm, referent, update?)` (coalescing upsert — only ever
  *raises* a name, never downgrades to null), `recall(realm, referent)`
  (O(1) point-get), `recallRealm(realm)`, `forget`, `forgetField`,
  `allBeliefs`, `loadBelief` (hydrate), `clearBeliefs` (evict).
- **The record spine** carries common fields (`realm`, `referent`,
  `knownAs`, `firstSeen`, `lastSeen`) plus a thin `payload`. **Payload
  rule: flag by default, value only for planned divergence.** `knownAs`
  is a *value* (faking/nicknames are planned). The identification realm
  stores a `typeKnown` *flag* — the type name is read live, never
  snapshotted.
- **Lazy liveness-GC**: `recall` drops a record whose referent no longer
  resolves to any live Stuff (`StuffApi.findAllByTemplatePath`).
- `_beliefs` is **not** a persistent field of the host — records are their
  own Documents (see Persistence).

## The compose seam — `RecognitionApi.describe(viewer, target)` (`api/recognition.ts`)

The viewer-aware naming step: `(viewer, target) → string`. The
consumer-intelligence layer over the dumb store.

> **Home note.** The requirements fix that this is **not** homed on
> `PerceptionApi` (it consults perception for its visibility gate but is
> its own concern — identity, not sensory channels). It lives on
> `RecognitionApi`, which also hosts the `learnIdentity` write-sink.

Algorithm (per target):

1. Baseline = `target.getPresentation()` (viewer-blind).
2. Not a `Sensor & Perception` viewer → baseline. Can't *see* the target
   (`VisionModality.canSee`) → the maximally-obscured form ("someone" /
   "something"), never the true-name baseline. (A backstop — real
   visibility filtering happens upstream in `look`/scope-walk.)
3. Masked (disguise) → baseline (which already reads as the covering's
   `appearsAs`); the known name is withheld by not consulting recognition.
4. **Recognition** (instance, living beings only): `recall(RECOGNITION,
   templatePath)?.knownAs`. **Identification** (type, any `Identifiable`):
   the known type name. The two **compose** — `"Bob, a city guard"`; each
   applies alone; an unknown being renders the **bare stem** — its
   `shortDescription` ("a crossing guard"), else a species fallback —
   never its true name.
5. `describe` is **pure identity**: no worn-feature affix and no
   status. The two escalations are separate surfaces — the
   distinguishing worn-feature form ("… wearing a faded hi-vis vest")
   lives on `salientFeatures` and rides `perceivedKeywords` for
   *targeting*; the activity-status affix rides
   `describeWithStatus` (below). Keeping both off `describe` is what
   makes ambient act lines read "a crossing guard says …", not the
   whole life story.

`describe` is **pure** — it runs for every perceived target × viewer on
every look / listing / MQL projection, so it never mutates memory. The
repeat-perception write fires on the perceive *controller* path, not here.

### Visibility gate — cycle avoidance

`describe` resolves `VisionModality.canSee` **lazily** via the registered
vision singleton's constructor (`StuffApi.findByTemplatePath` →
`.constructor`). A *static* import of any perception module into
`recognition.ts` would drag the `Modality → Idea` subsystem into eval
before `Idea` is ready (recognition is reachable from the root
`Stuff`/`Idea` graph via `Mml` and the MQL projection), crashing boot. So
`recognition.ts` carries **zero static perception imports**.

## The prose path — one central `Mml` hook

`Mml.actor/thing/player/npc/location(stuff)` produce a **viewer-aware
lazy fragment** (`{ kind: 'ref' }`) whose display text is bound *late* at
`toString(viewer)` time, calling `RecognitionApi.describe(viewer, stuff)`
— falling back to `getPresentation()` when no viewer (logs, `refOf`).

⭐ **`Mml.actor` binds its TAG late at the same seam**, through
`RecognitionApi.kindOf`, so a disguise hides not just a name but the
fact that a person is behind the figure. See
[messaging.md § The identity tags](./messaging.md).
Because `Scene.send` already materializes each frame body against its
recipient (`body.toString(recipient)`), **every scene line and look output
becomes per-recipient viewer-aware for free** — a broadcast names the same
target "Bob" to a friend and "a hooded figure" to a stranger, with no
change to the ~56 call sites. The client-data path
(`MqlSubscriptionApi.projectFields`) applies the same routine to the
universal `displayName` field, so the inspection pane and the scrollback
can't diverge.

## Recognition triggers — `introduce` + auto-introduce + repeat-perception

`RecognitionApi.learnIdentity(viewer, subject, name)` is the single
identity-learning sink (non-null name = introduction; null = a bare
sighting). All triggers funnel through it.

- **`introduce`** (`cmd/social/introduce.yaml` +
  `IntroduceController`): a social act, afforded by `SoulMixin` (on every
  Character), **NOT** `VocalMixin` — you can introduce yourself by speech,
  sign, or gesture, so it's modality-neutral and routes by perception, not
  earshot. `introduce` (self) or `introduce <subject>` (third-party —
  requires the actor already recognize the subject). The controller emits
  a public scene line on `act.deed` and, over the same
  recipient set (`MessageApi.getSensors(env)`), writes each recipient's
  record. **No content hook in the messaging substrate** — the controller
  owns the write.
- **Auto-introduce** (the dialogue build's ambient trigger): the shared
  `SoulMixin.introduceSelf()` runs the same introduce act (scene +
  `learnIdentity` to all in range), gated by
  `RecognitionApi.recognizes(viewer, subject)` so it doesn't re-introduce
  to someone who already knows the actor. Fired on arrival by the
  `introduces` NPC brain (intrinsic — social NPCs name themselves to
  newcomers) and by `Mobile.autoIntroduceOnArrival` for players who set
  `social.autoIntroduce`. So you learn the bartender's name by walking in,
  and speech then renders it. See [npc-dialogue.md](./npc-dialogue.md).
- **Repeat-perception**: `LookController` fires `learnIdentity(actor,
  target, null)` for each perceived being on the look chokepoint (never in
  `describe`). First sight creates a null-`knownAs` stranger record; later
  sightings coalesce and advance `lastSeen`.

## Disguise — `Disguisable` + `getDisguise` (`lib/disguise/`)

- `DisguiseBearingMixin` (worn-side, on a `Garment` → `DisguiseGarment`)
  carries a `Disguise` descriptor `{ appearsAs, covers, masksIdentity }`.
- `DisguisableMixin` (on `Creature`) — a **viewer-blind** `getDisguise()`
  resolver over worn `DisguiseBearing` garments (live slot-scan) + one
  transient imposed slot (`setDisguise`/`clearDisguise`, active-effect, not
  persisted). Merge = union `covers`, broadest `appearsAs`, any
  `masksIdentity`.
- **`Stuff.getPresentation()` defers to `getDisguise()`** — masking lives
  at the baseline, NOT a shadow on the synthesizer. The viewer-relative
  half (withholding a *known* name) is `describe`'s job.
- v1 reveal gate is dumb: `masksIdentity` ⇒ withhold the known name, render
  the covering. Per-channel partial recognition is v2. Removal re-fires
  recognition for free (the worn-scan simply finds nothing).
- Ships one content item — a **hood** (`covers: [face]`, slot `head`).

## Viewer-relative targeting — the name-leak gate (`api/mql/scope-walk.ts`)

Keyword resolution shares the naming step's source. `pushDirect` builds
each candidate's `name` from `RecognitionApi.describe(viewer, stuff)` and
its `keywords` from `RecognitionApi.perceivedKeywords(viewer, stuff)`
(tokens of the *perceived* name — `knownAs` if recognized, salient
features if not, the disguise's descriptors if masked; items keep their
ordinary keywords). The true name is never a keyword unless revealed, so
`look bob` resolves iff the room view shows the viewer "Bob".

- **Boundary**: the engine closes **direct** leaks (keyword resolution).
  It does NOT close **inferential** leaks (elimination, watching someone
  don a hood) — legitimate player reasoning.
- Ordinal disambiguation (`stranger:[2]`) is an index into the current
  candidate snapshot and carries no identity.

## `StatusMixin` (`lib/status/Status.ts`)

A settable activity-status line — "watching the empty road". Three
sources: the `status` verb, a runtime setter an NPC behavior pokes
(`setStatus`), and a static authored default (`authoredStatus`, the only
persisted field). Runtime overrides the default; clearing reverts to it.
Per-field invariant on the setter (collapse whitespace, reject over-long).
**Distinct from derived status-flags** (poisoned, glowing) — don't merge.

**A presence affix, not identity.** The status is *not* woven into
`getPresentation()` or `describe` (those are pure identity). It rides
`RecognitionApi.describeWithStatus(viewer, target)` — the presence-scan
variant used only by the room occupant roll-call ("a crossing guard,
watching the empty road"), never by act-subject naming ("a crossing guard
says …") or the look-at header. That split is what keeps the status from
contradicting the act in flight; see [message-rendering.md] /
`SocialLogic.composeOccupants` for the one presence consumer.

## Identification — type axis, thin (`lib/identification/`)

- `IdentifiableMixin` marks an item whose **type** is hidden until
  identified. Its presentation (`shortDescription`) is the *unidentified*
  appearance ("a blue potion"); `identifiedName` ("a potion of healing")
  is revealed by `describe` only to a viewer who has identified it. This
  **inverts** the recognition direction (a creature's baseline is its true
  name and `describe` hides it; an item's baseline is the unidentified
  look and `describe` reveals the type).
- Keyed on the item's `templatePath` (v1 keys on templatePath only;
  appearance-keying — one template, many appearances — defers).
- **One thin trigger**: a scroll of identify (`IdentifyScroll`) *carries*
  the `identify <item>` verb (inventory-bucket contribution — the object
  affords it, not a mixin on the actor). On read, writes
  `know(IDENTIFICATION, templatePath, { typeKnown: true })`. Binary
  (unidentified ↔ identified); reusable (no consumption). Ships a blue-vial
  demo. The masking mechanism supports item-identity *illusion* by design
  but no illusion content ships.
- **`payload.believedName` — a name the viewer holds and that may be
  FALSE.** The store is a record of what someone *thinks*, and until this
  existed nothing exercised that: every record was true-or-absent. A
  cursed identify plants one — a record indistinguishable from a real
  identification, naming a *different real thing*, so the lie is
  plausible. Absent ⇒ the referent's own `identifiedName`, i.e. the truth.

  Two consequences worth stating, because both are load-bearing:

  > ⚠ `know()` assigns `believedName` **unconditionally whenever
  > `typeKnown` is written**, rather than guarding it like its
  > neighbours. A type-learning act *replaces* what you believe the type
  > is, and an honest identification omits the field — so omitting it has
  > to CLEAR a planted name. Guarded, a curse would be permanent, and
  > finding out is the whole of how a holder recovers.

  > ⚠ It also gates the **prose**, via `RecognitionApi.knowsTrueType`: a
  > misidentified item keeps its class's generic long description rather
  > than revealing the authored one, which would contradict the planted
  > name in the very next line. See
  > [magic-items.md](./magic-items.md) § *The class owns the
  > unidentified paragraph*.

## Regard — attitude axis, the third realm (`api/regard.ts`)

A **third belief realm** (`REGARD`), a sibling of recognition and
identification: the viewer's *attitude* toward a subject — how much it
likes / trusts / esteems them — stored as one **signed scalar**
(`payload.regard`, `-100..+100`; absent or `0` = no opinion) per directed
`(viewer → subject)` pair. Keyed by the subject's `templatePath`, exactly
like recognition — "the same Bob, three realms at one referent key": a
separate record from recognition's, not a field on it. Regard is the
per-viewer leg D&D charisma unbundles into (regard + renown +
susceptibility — see the reputation slate) and the *reciprocation /
social-embeddedness* primitive of the future reputation / Compact-
polity Sybil keystone.

- **Overwrite, not raise-only.** `know` *raises* `knownAs` and merges
  flags, but for `regard` it **overwrites** the scalar — the read-modify-
  write delta arithmetic and the normative clamp live in the consumer,
  never in the dumb store.
- **`RegardApi` / `RegardLogic` — the gated arithmetic seam.** `api/regard.ts`
  is a thin forwarding shell to the hot-reloadable `RegardLogic` singleton
  at `/obj/api/regard` (`StuffApi.singletonSync`; methods gated
  `FromModule('/api/regard#RegardApi')`), mirroring
  `RecognitionApi`→`RecognitionLogic`. Surface: `getRegard` (absent → 0),
  `adjustRegard` (the clamped accumulator), `setRegard`, `clearRegard`
  (`forgetField`), `regardsHeldBy` (a `recallRealm(REGARD)` projection).
  The clamp + the eventual read-time decay (off `lastSeen`) live here; the
  store stays dumb CRUD.
- **Kind-agnostic edges.** The substrate stores no player/NPC marker and
  the same path serves player↔player, player↔NPC, NPC↔player, NPC↔NPC.
  The player/NPC distinction (trust-weighting, susceptibility) is a
  deferred *consumer* concern, never the edge's.
- **Orthogonal to `ContactsMixin`.** Contacts is *categorical* (which
  named list — `friends`/`foes`); regard is *scalar* (how much). Separate
  layers, bridgeable but not merged (a `foes` membership may later *seed*
  negative regard); see [contacts.md](./contacts.md).
- **Demo mutator.** `IntroduceController` warms each in-earshot recipient
  toward the introducee by a small bump — illustrative, riding a moment
  that already fires (the chronicle's demo-minter discipline). The real
  triggers (thanks / gifts / slights, NPC reactions) land with
  reputation / npc-behavior.
- **Deferred consumers (none built here).** Renown aggregation, per-circle
  vectoring, trust-weighting / eigenvector, social-graph display, NPC
  susceptibility, decay tuning, and verbs all project *from* this scalar
  later. The substrate ships designed-for-them but ships none of them.

## Discovery — the world-fact realm, the fourth realm (`DISCOVERY`)

A **fourth realm** (`DISCOVERY`), the belief store generalized from
*identity* memory to **world-facts** — "viewer V has found feature F." It
is the place-memory that belief.md long named as a future realm, shipped in
its **world-fact cut** (a found *feature* — a secret door, a hidden cache, a
trap — not room familiarity, still distinct). Added by the concealment build
(see [concealment.md](./concealment.md)):

- Payload is a bare `found?: boolean` flag (flag-by-default — no value
  divergence, unlike identification's `knownAs`).
- Keyed on the concealed thing's **discovery key** — a `Concealable`'s
  `getDiscoveryKey()`, defaulting to its `templatePath`, but an `Exit`
  (which carries none of its own) keys on a synthetic
  `` `${source.path}#exit:<dir>` `` handle so a secret door stays
  discovered across re-clones.
- Written only through `PerceptionApi.recordDiscovery` (`know(DISCOVERY,
  referent, { found: true })`) and read through `hasDiscovered`
  (`recall(...).payload.found`) — the presence face owns the sink, not
  `RecognitionApi` (the identity face).
- **GC-exempt.** The lazy liveness-GC that reaps records whose referent is a
  dead Stuff is bypassed for `DISCOVERY` — its referent can be an `Exit`'s
  synthetic key with no live Stuff to resolve, so reaping it would forget a
  valid discovery. A found-flag is a cheap, durable, per-viewer world fact.
- Per-viewer isolation, no-inherit, and persistence (`beliefs` collection)
  are inherited from the store unchanged.

## Persistence — lazily-hydrated working set (`api/belief.ts`)

`BeliefStoreApi` over `BeliefDocument extends Document` — a dedicated
**`beliefs`** collection, one document per `{viewerId, realm, referent}`,
indexed on `viewerId` **and** on `{realm, referent}` (both declared
centrally in `PersistenceManager.createIndexes`). The `viewerId` index
serves the forward direction (a viewer's lazy-hydrate + cleanup cascade);
the `{realm, referent}` index serves the **reverse** direction — "all
beliefs held *toward* subject X," the regard realm's renown / Sybil-
keystone data path (no consumer reads it yet). NOT one-big-doc-per-viewer
(the
`ContactsMixin` anti-precedent — 16MB cap, whole-array rewrites). Goes
through the `Document` wrapper (`find`/`save`/`delete`), not raw Mongo;
upsert keys on `{viewerId, realm, referent}` via a find-then-save (a read
on the *write* path — never the naming path, so the no-read constraint
holds; sequential single-viewer commands keep the race benign).

- **Lazy hydrate** on `Avatar.enter`; **evict + final-flush** on
  `Avatar.onDestruct`; **per-record write-through** fired fire-and-forget
  from `know`/`forget` (inert when Mongo is closed — tests, pre-boot).
- **Write-through gate**: only a record that has *learned* something
  (`knownAs` set, or a payload flag) persists; bare null-`knownAs`
  strangers stay session-local.
- **No Mongo read on the naming path** — `recall` is pure in-memory; Mongo
  is touched only on hydrate + write-through.
- **NPC viewers**: durable-`templatePath` NPCs (named / singleton) persist;
  generic clones are session-ephemeral by construction (no durable key) —
  falls out of the keying.
- **Cascade-ready, not cascade-owning**: owner-keyed + `viewerId`-indexed
  so a future account-deletion cleanup cascade (`deleteMany({viewerId})`
  on an account `aroundDelete`, plus a liveness-GC backstop — GDPR/erasure)
  can purge it. The cascade itself is the persistence layer's job; no
  account-deletion hook exists to wire today.

## Deferred tails

- **Aether id-aug + anonymity** (deferred) — an `AugmentMixin` broadcast over
  the aether to attuned receivers, calling the same `learnIdentity` sink,
  anonymity via an `identity.broadcast` setting. **Not built**: the design
  axes (reception attunement-vs-innate, disguise orthogonal-vs-pierce) are
  unresolved AND the augmentation substrate is itself being retooled. The
  explicit `introduce` verb is the shipped core trigger.
- Identification's **pedagogical instrument seam** (`analyze X with Y`,
  real Material-substrate chemistry), **partial identification**,
  experience-/social-ID verbs. (**Misidentification** is no longer
  deferred — `believedName` above ships the belief-vs-truth split and the
  cursed identify scroll plants one. Illusion *content* still doesn't
  ship, but the mechanism it would ride does.)
- **Place-memory** — the *feature-discovery* cut shipped as the `DISCOVERY`
  realm (above); *room-familiarity* place-memory is still a future realm,
  alongside the shipped recognition / identification / regard / discovery
  set. Social-graph crowd verbosity,
  player-set **nicknames**, memory **decay**, voice/scent recognition, MQL
  compound feature-handles (`talk to tall-stranger`).

> **Stale doc to correct (perception.md):** the `RecognitionShadow`
> example in `perception.md` (and the recognition-slate's
> `getPresentedIdentity`-shadow design) is superseded. Recognition is NOT a
> Shadow — it's the explicit `RecognitionApi.describe` entry point;
> disguise is NOT a shadow on the synthesizer — it's `getPresentation`
> deferring to `getDisguise`.

## Naming across the sandbox boundary

`RecognitionApi.describe` / `describeWithStatus` / `perceivedKeywords`
/ `salientFeatures` route through `SecurityApi.projectAcross` (see
[call-security.md](./call-security.md)). `describe` is the one place
the engine answers "what does THIS viewer call THAT thing", and both
halves routinely sit on opposite sides of a circle: a channel post from
the field renders for a recipient inside one; `who` from inside renders
every field person for a viewer who isn't. The walk (perception, light,
disguise, belief) then reads the far object and the boundary denies it.

Naming is exactly what the doctrine already permits to cross — a pure
read yielding text — so the projection runs under an omni root when,
and only when, the two are on opposite sides.

One consequence worth carrying: **an unlit room makes its occupants
unnameable.** `canSee` fails at `pitch-black`, `describe` falls back to
"someone" / "something" for everyone present, and `tell <name>` cannot
resolve a target at all. See [sandbox.md](./sandbox.md).
