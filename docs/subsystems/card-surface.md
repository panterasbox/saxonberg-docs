# The card surface

The right column is a **feed of cards**, and a card is one thing: *a
container for structured content, separate from the unstructured
message feed in the terminal*. It is not a lifetime system, not a
liveness system and not a layout system — everything here exists to
make it that and nothing more.

## ⭐⭐ One birth path

**A card exists because a COMMAND caused the server to push it.** The
client no longer infers a card from a changed query result, and it
cannot ask for one either.

That is enforced three ways, strongest first:

1. **The protocol.** `MqlSubscribeMessage` carries no field that could
   name a card. The one subscription a client may still open for itself
   is `chrome: 'self'` — the widget shelf's figures, which is *not a
   card* (no pinned-ness, no lifetime). A source scan can be defeated by
   a clever call site; a missing protocol field cannot be used at all.
2. **The gate.** A command view declares `opens_card:` (a string, or a
   list where a verb legitimately opens more than one). `CardApi.open`
   reads the running command and **throws** when it opens a card the
   view did not declare — so the vocabulary stays declarative and
   greppable while the call site stays where the resolved operand is
   known. Validated against `CARD_IDS` at **load**, so a typo fails at
   boot rather than at first invocation.
3. **`card-birth-path.test.ts`**, which asserts every mint site and the
   one client writer *by name*, as a set.

The predecessor was the opposite of this: a card was born when the
client noticed the standing `inspect` subscription's first result had
changed — *"the `inspect` subscription is the SIGNAL, not the card."*
Three things made that untenable. A verb like `who` had no way to open a
card at all; the server could not assign a lifetime to something it did
not know existed; and the authoring surfaces have no focus change to
hang off.

## ⭐ Two axes, independent, both REQUIRED

```ts
export interface CardDefinition {
  readonly label: string;
  readonly source: CardSource;
  readonly pinnedByDefault: boolean;   // ⚠ required
  readonly live: boolean;              // ⚠ required
  readonly command: string;
  readonly noProse?: true;
  readonly singleton?: true;           // re-running touches, not stacks
}
```

**Pinned is the whole lifetime.** An unpinned card ages out of a
relevance window since it was last touched, and **says so** when it
goes. A pinned card stays until dismissed. The catalogue declares the
default; the player overrides in both directions with `cockpit card
pin | dismiss | auto`.

**Live is opt-in in the catalogue and SCOPED BY ATTENTION at runtime.**
A card is **static by default** — resolved once, stamped with
`takenAt`, carrying a refresh control. A live card carries neither,
because *a static card that looked live would be a lie*, and a refresh
button on a live card is a bandage over a wake that does not fire —
**and, worse, it is how nobody finds out**. A `here` card was immortal
through eleven passing tests because every one of them called
`refreshForInteractive` by hand.

⭐⭐ **Only the NEWEST card of a live kind actually holds a
subscription.** Opening one calls `CardRegistry.demoteLive`, which
tears the predecessor's handle down, stamps its `takenAt`, marks it
`demoted` and tells the client (`card-touched { live: false }`). From
that moment it is an ordinary snapshot with a refresh, like every other
static card.

> *"Liveness is expensive so we don't want it on anything more than
> where you're currently focused."*

That sentence is the design, and newest-card-is-live is a crude but
honest proxy for it. It is also why liveness **cannot** be what tells
two catalogue rows apart: it is not a property of a card KIND at all,
it is a property of where the player is looking. Behind you there is
nothing to track, and a subscription still running would be a wake with
no reader.

⚠⚠ **What "live" costs, and the four ways it was quietly dead.** The
one live row shipped not-updating, and each of these alone was enough —
none of them visible to a green suite:

1. **The delta was unapplicable.** A `cardinality: 'one'` answer that
   moves to a different subject emitted a lone `replace` under a key the
   consumer had never seen; every consumer looks its record up by
   `stuffId`, misses, and **appends**. The old room survived at index 0.
   `MqlSubscriptionRegistry.diff` now emits the `remove` beside it —
   *the answer moved, so it is a removal and an arrival*. ⚠ Two
   consumers had private bypasses for this already, which is why it
   outlived them; the generic path everything else uses still had it.
2. **Touch re-asserted the birth body.** A live card's `records` are its
   first resolve and are never written again, so a bare `look` pushed
   the room you LEFT back over a client that already knew better.
   **Whoever owns the body owns every update to it**: a live card's
   touch carries none.
3. **The name was frozen at open.** `card.title` is stamped once; the
   body names the card, and `title` only fills in until it arrives.
4. ⭐⭐ **The query was RELATIVE, which is the one that mattered.** The
   room card resolved `here`, and `here` re-answers against **whoever is
   asking** on every wake. Leaving the lounge changes the lounge's own
   contents (you left), that wakes the card, and the card re-answers
   `here` — which is the bar. *The lounge card silently became a bar
   card*, and dropping `locationDependent` did not help, because the
   room's own contents were enough to trigger it. **A relative query can
   never back a card about a THING.** Closed by binding the subscription
   to a subject (below).

The lesson under the first three: *asserting that a delta was SENT is
not asserting that a card updated.* The test now applies the changes by
the wire's own rule and asserts the result. The lesson under the fourth
is bigger and outlives this subsystem: **a card is about a subject, so
everything that re-derives its content must be anchored to that subject
— the query, and the refresh command too.**

All four combinations are meaningful, and neither axis implies the
other. Both fields are non-optional in a `Record<CardId,
CardDefinition>` — the `COLLECTION_POLICIES` trick from `ResetPolicy` —
so **a new card without a lifetime decision is a compile error**. That
is what makes "a new card cannot ship without choosing its lifetime"
enforceable rather than aspirational.

### What the five holds became

The predecessor had five: `here`, `present`, `inReach`, `carried`,
`unanswered`. The four spatial ones bought precision (a card closed the
moment you left the room rather than eventually) and each cost a wake to
fire. That trade is **declined**: you can scroll back, and the card ages
out.

⚠⚠ `unanswered` is not symmetric with the other four and does not
simply die. Its subject was a pending **command**, and it is the one the
design leans on — *nothing that is still actionable ever leaves*. Its
guarantee moves onto the pinned axis: a **prompt card opens PINNED and
auto-releases with reason `answered`** when the prompt settles
(`PromptLogic.cleanup` → `interactive.notifyPromptSettled`). Same guarantee,
one axis, no hold vocabulary.

## ⭐ One identity: the normalized command

A card is identified by the command that produced it —
`CardLogic.normalizeKey`, three rungs:

1. `ShellApi.expandAliases` — the player's own aliases.
2. **Canonicalise the verb to `command.verbs[0]`.** ⚠ `examine` is a
   verb SYNONYM, not an alias (`look.yaml` declares `verbs: [look, l,
   examine, exa]`), so `expandAliases` never sees it. Without this rung
   `examine a` and `look a` are two cards.
3. `CommandLineApi.format()` — the canonical round-trip.

⭐⭐ **The key is the card's IDENTITY and its REFRESH — it is not a
uniqueness constraint.** The feed is a **log**. Asking twice mints two
cards, because you asked twice and the feed records what you asked; a
transcript does not dedupe and neither does this. Only a
{@link CardDefinition.singleton} row reuses.

| Sequence | Result |
|---|---|
| `who`, `who` | **two** cards, the second below the first |
| `who`, `who --wizards` | **two** cards, with different keys |
| `look a`, `look b`, `look a` | **three**; two of them keyed `look a` |
| `examine a`, `look a` | **two** cards, **one** key — the synonym rung |
| `cms`, `cms` | **one** — a singleton row |

⚠⚠ **Dedup-on-command was the model until it was driven, and it failed
in the worst available way.** Asking again changed nothing visible — and
when the touched card was already on screen its prose was suppressed as
`carded` too, so the command appeared to do *nothing at all*. `look
dave` in Dave's Bar printed its echo and stopped. The singleton
exceptions are the surfaces that ARE one thing — an editor, a git
panel, a composer — because a second Monaco with its own unsaved buffer
is not a second reading, it is a second application.

⚠ **A knowing cost:** `look lamp` and `look brass lamp` are two cards
about one thing. That is identity-is-what-you-typed read literally, and
it is the right trade — but it is a cost, not a defect.

**Ordering.** The feed runs oldest → newest, like the transcript beside
it, and cards do not reorder. **Pinned cards hold their position** too;
pinning changes only whether the sweep may take a card, not where it
sits.

⚠⚠ **The refresh key derives from the command, and forcing it was a
bug.** The room card used to be given `place`'s catalogue command —
bare `look` — as its key, so that a typed `look` would touch the
arrangement-pushed card rather than duplicate it. Two things killed
that: the log model removed the premise, and bare `look` is a
**relative command**, the same trap as the relative query in the axis
section above. A card about Dave's Bar keyed on `look` refreshes to
*wherever you are now*. Every inspection card now takes the key the
parser composed from what was actually asked, and
`card-identity.test.ts` checks the call sites at the source — a forced
`key:` growing back would not fail a behavioural test.

## The catalogue — three sources

MQL speaks **Stuff**. The roster is `RosterRow[]`, releases are `Release`
documents, a wiki page is a rendered payload and the authoring surfaces
are REST. An MQL-only catalogue could not express three of the ten
shipped rows at all, so `CardSource` is a discriminated union:

- **`mql`** — a server-owned query resolved to Stuff and projected. The
  client sends no MQL. A `needsSubject` row resolves by direct lookup
  **behind the perception gate**; ⚠⚠ never an `#<stuffId>` MQL seed,
  which is authoring-tier and ungated, so a card built on it would
  answer for anything whose id the viewer had ever seen on a frame — a
  peep-hole into every room in the game, looking exactly like the
  feature working.
- **`payload`** — a `CardPayload` **the producing controller already
  computed**. ⭐ `WhoController` builds `RosterRow[]` to render its
  prose; the card carries that same array. A producer on the card side
  re-deriving it would be two computations of one answer — the
  two-copies-of-one-sentence shape at the level of data rather than
  words.
- **`client`** — the body is the client's own transport (Monaco, the git
  panel, the Studio catalogue). The **server** still owns the card's
  existence, identity, lifetime and pinned-ness; only the body is the
  client's.
- **`prompt`** — no body at all. The client already holds one prompt
  model and the card joins it by `promptId`.

  ⚠ **Pushed by `PromptApi`'s own push path**, not by a command — a
  question is the one card nobody types a command to get. Its key is
  `prompt <promptId>`, so two questions waiting at once are two cards;
  keyed on the bare command the second would TOUCH the first and
  silently retarget it. Being a question rather than a reading, it
  carries **no refresh control** even though it is static.

  ⚠⚠ **This push was missing for a whole build.** The row, the settle
  path, the client body and five green tests all existed; the card push
  had **zero production callers**, and every test hand-pushed the card
  it then asserted on. `card-birth-path.test.ts` now asserts the other
  half of the question — *is every declared kind reachable* — beside the
  list of mint sites it already had.

The nine shipped rows:

| id | source | live | pinned | key | prose |
|---|---|---|---|---|---|
| `subject` | mql `$subject`, `detail` | **✓** | ✗ | `look <subject>` | `look`'s / `sense`'s body |
| `who` | payload `roster` | ✗ | ✗ | `who` | `WhoController` |
| `news` | payload `releases` | ✗ | ✗ | `press` | `PressController` |
| `wiki` | payload `wikiPage` | ✗ | ✗ | `wiki <slug>` | the page read |
| `help` | payload `helpTopic` | ✗ | ✗ | `help <topic>` | the topic read |
| `prompt` | prompt | ✗ | ✓ | `prompt <promptId>` | `noProse` |
| `cms` | client | ✗ | ✓ | `cms` | `noProse` |
| `git` | client | ✗ | ✓ | `git` | `noProse` |
| `studio` | client | ✗ | ✓ | `studio` | `noProse` |

### ⭐⭐⭐ Inspection is ONE row, not two and not four

Looking at something mints a card laid out by what the subject **is** —
`location` · `agent` · `thing` · `idea`, the four top-level Stuff
branches, carried per-card as {@link StuffKind} and dispatched on by the
body. Going narrower (a weapon vs a lamp, an NPC vs a player) waits
until something forces it.

**Those four are kinds of one card's subject.** The catalogue carried a
second row — `place`, for rooms — and the tell was a command view that
had to read:

```yaml
opens_card: [place, subject]
```

`look` takes one target; that target is by construction exactly one kind
of thing. A verb declaring that it opens *one of two kinds of card* is a
verb reporting that the model is not unified. By then the two rows had
`source` blocks identical field for field (`$subject`, `cardinality:
'one'`, `fields: 'detail'`, `needsSubject`), and every difference that
survived was either a **lifetime** decision (does it start live) or a
**layout** decision (does it show exits) — neither of which is an
identity, because lifetime is decided by attention and layout by
`StuffKind`.

⭐ **A declaration site is where an un-unified vocabulary becomes
visible.** The two rows read fine in the catalogue, where they sat
apart; forced to name themselves on one verb's spec they were a list of
two. When checking whether a vocabulary is really one concept, read the
DECLARATIONS, not the definitions.

⭐ The evidence the second id was surface area rather than structure:
after cutting it, **every production file compiled clean** and only
tests failed.

⚠ `subject` is therefore the one live row, and it is the one that earns
it: it is the card the player is looking at, which is the whole basis on
which liveness is granted. **`who` ships static** for a reason that has
not changed — the dependency vocabulary is `focusDependent` /
`locationDependent` / the `durableKey` poke channel, and **nothing wakes
on connect or disconnect**. A live `who` would resolve once and then be
permanently wrong while looking exactly like it worked.

⚠ `CardSource` no longer offers `locationDependent` at all. The one row
that ever set it was the room card, which is exactly the card it broke
— *re-resolve when the viewer moves* is a wake for a relative query, and
an inspection card is bound to its subject. The wake still exists for
the subscriptions that want it; a catalogue field no row sets is a field
nothing can be caught getting wrong.

⚠ A catalogue where every row was static would leave `live` a field
nothing reads, and a declaration nothing reads is indistinguishable from
a broken one.

### ⭐⭐ `self` is NOT a card

The widget shelf's subscription lives beside the catalogue as
`SHELF_SUBSCRIPTION`, explicitly outside `CARDS`. It has no pinned-ness
and no lifetime, and forcing it to declare them would make the
required-fields gate meaningless — which is the gate that makes the
whole "strict taxonomy" claim real. The client names it on the wire as
`chrome: 'self'`.

## ⭐ One sweep, and exactly one clock

Lifetime is a **relevance window since last touched**, evicted by ONE
recurring pass over the whole set — the `ResidencyLogic` shape, through
`ScheduleApi.recurring`, never a bare `setInterval` and never a timer
per card.

```
for interactive, card in cardSet:
  if card.pinned: continue                 # pinned is the whole axis
  if now - card.lastTouchedAt < windowMs: continue
  close(card, 'aged-out')                  # → card-closed, which the husk renders
```

⚠ **The client's own husk `setInterval` is deleted.** Two clocks
disagree, and the one the player cannot see is the one that wins
arguments. The server owns the window, so `cockpit card list` and the
client agree by construction.

⚠ The cadence is not the window. A coarse sweep (~30 s) with a fine
window is the residency shape; conflating them means changing one
silently changes the other.

⭐ **The window is the `cards.window` SETTING** (600 s by default), read
per player *inside* the sweep — one sweep, many windows, which is what
lets a player shorten theirs to watch a card age out and say so without
changing anybody else's. There is no constant beside it: a number with
two homes would have the one the player cannot see quietly in force.

⚠ It is a legitimate duration because it is **a fact about TIME, not
about the world**. A card's lifetime used to be a world condition, and a
clock on one of those ends something still actionable; a relevance
window is the husk-TTL argument (*"the one legitimate duration in the
card model"*) generalised.

⚠ **The scheduled callback re-plants the principal.** It fires long
after the frame that installed it, so the execution context has no
target and the registry's gate would deny every tick — *silently*,
because a scheduled callback has nobody to report to. `CardApi.boot`
wraps it in `ExecutionContextApi.runRoot` with the logic singleton as
principal.

## ⭐⭐ Subject-bound subscriptions

A live inspection card does **not** run its `query`. `'$subject'` is an
inert marker; `needsSubject: true` says *this card is about a
particular thing*, the open supplies a `stuffId`, and the subscription
resolves it by direct lookup.

```
SubscribeRequest { subjectId }          →  MqlSubscriptionRegistry
  resolveSubjectBound(subjectId, viewer):
    found = StuffApi.findById(subjectId)
    return PerceptionApi.perceives(viewer, found) ? [found] : []
```

⭐⭐ **Perception is re-checked on EVERY re-resolve, not once at
subscribe.** A `stuffId` is not a capability. A card that gated at
birth and then trusted the id would keep answering for a subject that
had since been concealed, moved behind a door, or disguised — a
peep-hole that looks exactly like the feature working. This is the same
reason the row may not use an `#<stuffId>` MQL seed: that seed is
authoring-tier and ungated.

⚠ **Why it exists at all** is the fourth liveness failure above: a
relative query (`here`, `$focus`, `person`) re-answers against the
**asker**, so any wake re-points the card at the asker's current
situation. A card about a THING has to be anchored to that thing, and
nothing about the query text can express that — the anchor has to be
the subject id, carried on the subscription.

## ⚠ No migration ships, and here is why

This build renamed the stored key (`cockpit.panes` → `cockpit.cards`),
re-keyed `SHIPPED_ARRANGEMENT_CARDS` by (mode, arrangement), retired the
five holds and cut a card id — every one of which would normally want a
migration for the players already carrying the old shape.

None is written, because **`cockpit.arrangements` and the rest of the
cockpit keyspace live on `holder_snapshots`, which `ResetPolicy` wipes
nightly.** A migration would run against rows that do not survive the
night; writing one would be ceremony that could never be observed
working or failing, which is worse than none.

⚠ **The condition, not the conclusion, is what to check when this
changes.** The moment any of this keyspace becomes durable — a
`keep`-policy collection, an export, a per-account rather than
per-character home — the argument expires and the migration becomes
real. It is the reset policy that makes it unnecessary, not the shape of
the change.

⚠ **One thing is NOT covered by that argument and needed a hand step:**
the seeded `CockpitPaneController` template row in `domain`, because
`SeederManager` (since retired) was insert-only and `domain` was a `keep` collection. It
is `db.domain.deleteOne({path:
'/platform/idea/cmd/shell/CockpitPaneController'})`, run by hand; the seeder
now prints the general case's `deleteMany` at boot but does **not** run
it, because CMS-authored templates share that collection with no
discriminator.

## ⚠ The structural call: a second registry

The card set does **not** live inside `MqlSubscriptionRegistry`. The
earlier argument — *"the card set IS the existing subscription registry,
and there is no second registry to drift"* — rested on every card being
a subscription. After this build most cards are static, `payload` and
`client` sources are not MQL at all, and a prompt card never was; the
identity is broken, and keeping them fused would put non-MQL state in
the MQL substrate.

The drift risk therefore **moves** to *card ↔ its subscription handle*,
and is closed by making the card **own** the handle:
`instanceId === subscriptionId` for a live card, every teardown through
`CardLogic.close`, and `card-subscription-orphans.test.ts` asserting
zero orphans after closing, sweeping, rearranging and disconnecting.

⭐ It is also what lets a live card's updates keep riding
`mql-subscription-delta` — no new envelope, no join table.

## The tiers

| File | Role |
|---|---|
| `mud/lib/connection/Cards.ts` | The catalogue: `CardSource`, `CardDefinition`, `CARDS`, `CARDS_BY_NAME`, `SHELF_SUBSCRIPTION` |
| `mud/api/card.ts` | `CardApi` — the gated face; owns the sweep handle and the `runRoot` principal |
| `mud/platform/idea/api/CardLogic.ts` | The hot-reloadable logic singleton at `/platform/idea/api/card`; `normalizeKey` lives here |
| `mud/platform/idea/CardRegistry.ts` | The state: per-Interactive open cards, the sweep, `resolveSubject` behind the perception gate |
| `packages/client/src/store/cardFeedSlice.ts` | The client's card set + the husk model |
| `packages/client/src/components/cards/` | `CardFeed`, `Card`, `CardBodies`, `CardViewStrip`, `useCardFeed` |

⭐ `dest /platform/idea/api/card` reloads the logic **without closing anybody's
cards**: the state is on the registry, the resolution is on the logic.

## The wire

Four card envelopes plus the delta a live card rides:

- **`card-opened`** — `instanceId`, `cardId`, `key`, `live`, `pinned`,
  `takenAt?` (static only), `title?`, `subjectId?`, `promptId?`,
  `prose?`, `result?` / `payload?`. **Every** card arrives this way;
  there is no `pushed` flag, because the distinction it drew is gone —
  the client never opens a card, so every open is a push.
- **`card-touched`** — the same command re-issued.
- **`card-closed`** — `instanceId` + `reason`.
- **`card-pinned`** — the override took.

```ts
export type CardCloseReason =
  | 'answered'    // a prompt card settled
  | 'aged-out'    // the relevance window lapsed — it SAYS SO
  | 'dismissed'   // the player dropped it
  | 'rearranged'  // the workspace changed, not the world
  | 'gone';       // a live card's subject stopped existing
```

⚠ **A card that vanishes without a reason reads as a bug, and the
player cannot tell a rule from a defect.** Timing out is a reason and
must be said, which is why `aged-out` exists at all rather than the card
simply disappearing. The husk model carries over unchanged: fade, state
the reason, clear the body, keep the title.

⚠ **The title survives; the body does not.** Rendering yesterday's
contents as if they were current is the failure the fade exists to
avoid — but the title is *which card this is*, and a husk that cannot
say what it was about is less honest rather than more. Found by driving:
teleporting out of the lounge left a card naming nothing at all.

## Named views over the feed

`CardViewStrip` + `store/cardViewActions.ts` mirror the terminal's
`TabStrip` / `consoleActions` — same gestures, same vocabulary. Views
filter on **card kind**, the one axis a player can name.

⭐ **Two STRUCTURAL views ship: `All` and `Look`.** Neither is stored,
neither can be renamed or deleted, and neither opens an editor.

- **`All` is the absence of a filter**, not a stored row. That is what
  makes it locked and undeletable *rather than merely not currently
  deleted*, and it is why deleting every view a player made is safe.
- **`Look` is `INSPECTION_CARD_IDS`** — the cards born of looking at
  something, which is the one grouping a player can name without being
  taught it. It is a shared constant rather than a hardcoded list, and
  `card-catalogue.test.ts` asserts it is exactly the catalogue's
  `needsSubject` rows, so the tab and the catalogue cannot drift.

⚠⚠ **Fixed tabs are the LAUNCH shape, not the intended one.** What this
wants is **tagging** — cards carrying tags, views composed from a tag
library — and the confusing part of what ships is the `+` that makes an
empty view. Two named tabs are enough to launch on and the substrate
underneath is the same either way; the tag library wants a fuller set of
card kinds to form around, which does not exist yet.

⚠ And a consequence worth stating plainly: with inspection collapsed to
one card kind, **`Look` and `All` show nearly the same thing in ordinary
play**. That is not a bug in either tab — it is what happens when a
filter axis has one populated value, and it is more evidence for tags
than against them.

⚠ **Rename is INLINE**, like `TabStrip`'s — not a `window.prompt`. A
native modal is a different gesture in the one place the two strips were
meant to be indistinguishable, and the phone pays for it.

⚠⚠ **Selecting a view closes any other view's editor.** Without that,
switching away left the previous view's editor open beneath the new
selection: an unlabelled panel of kind toggles plus `rename` and
`delete`, acting on a view the strip was no longer highlighting. It is
the same failure as two cards sharing one pin command — *a control that
does not act on the thing it appears to be on* — and worse, because one
of the controls is `delete`. Both found by driving.

⚠⚠ **The seeding clobber, closed one step earlier than the fix it
learns from.** `console.tabs` shipped a bug where an ABSENT key read as
*first run*, so a layout mounting before the connection payload wrote
ship defaults over saved views — fixed by keying the seeding effect on
`Array.isArray(...)`, with **no test ever written for it**. The card
views ship **no defaults at all**, so there is no write to race.
`readViews()` still distinguishes *absent* (`null`) from *empty*
(`[]`), because a caller that could not tell them apart would
reintroduce the same question. Both halves are asserted.

⚠ `activeCardKinds()` returns `null` for `All`, not "every kind": a view
listing every kind and the absence of one look identical until a card
kind is added, at which point the enumerated one silently stops showing
it.

## `shell.result` — a FILTER, not a placement

One setting: `card` (default) · `terminal` · `both`.

**Why a filter.** Placement (the server declining to send) saves the
wire, but the frame then never reaches the frame store and **`recall`
cannot find it**. Filtering keeps your `who` history searchable while
keeping it out of sight.

⚠⚠ **It keys on `meta.carded`, not on a topic — and that is a change
from the requirements' decision 10.** That decision keyed it on the
topic `shell.result`, on the premise that *every structured command
result already carries it*. The per-card prose audit falsifies the
premise: `look`'s two cards ride **`sense.survey`**, which twelve other
verbs share (`get`, `drop`, `inventory`, `wear`, …). A topic key would
either miss `look` entirely or silence all twelve. The marker is exact
by construction — the producer that opens the card is the producer that
stamps the frame.

⚠ Under `terminal` the **card** is suppressed instead — **except where
it declares `noProse`**. A Monaco editor has no terminal rendering, so
suppressing it would take the authoring surface away on a setting that
never claimed to. The absence of `prose` on the wire IS that
declaration arriving.

⚠ **A backfilled frame is never filtered, and that is a decision.** The
filter hides a frame because a card is showing the same content *right
now*; a backfilled frame has no card — it is your history — so hiding it
would lose exactly what the filter promised to keep. Found by driving:
the previous session's `who` prose renders in the transcript while the
live one does not, which is the correct asymmetry and reads as one until
you know why.

⭐ **`both` is safe because there is only one rendering.** The card
carries the same `Mml` the controller emitted, materialized once against
the same viewer with the same `Mml.toString(recipient)` the Scene
composer uses. So the test asserts `frame.body === card.prose` —
literal equality of one payload, never the words twice — and `terminal`
is a first-class mode for free rather than a second renderer.

### The per-form-factor override

`SettingsSchemaEntry.perFactor` + `ShellApi.resolveSetting(host, key,
factor?)`: `<key>.<factor>` → `<key>` → schema default. `setSetting`
accepts a suffix **only** on a `perFactor` entry and only for
`desktop`/`mobile` — that is what makes it *one key with an optional
override* rather than an open namespace.

⚠ Rung 1 reads `getOwnSetting`, not `getSetting`. `getSetting` falls
back to the schema default, so a suffixed read would always return
something and rung 2 could never be reached: the override would silently
become **mandatory**, which is the two-independent-keys shape this
design refused.

⚠⚠ **This does not break the no-`cockpit.formFactor` rule.** That key
was never built because the server cannot know a viewport, so it would
be a fake fact. Two stored *preferences* assert nothing about which is
in force: the server ships both resolved answers on
`ConnectionEstablishedPayload.resultDisplay` (re-pushed as
`client-state-update` on every write, so the setting takes effect
without a reconnect) and the client — which genuinely knows its own
width — picks. Same split as `cockpit.shelf`.

## The card's action row

`AffordanceEntry.source: 'subject' | 'actor'`. The row renders
`'subject'` only.

It shipped showing `cast · defend · destruct` on everything, because the
entry could not tell *the actor can always do this* from *this subject
affords it*. ⭐ The resolver **already computed** the distinction
(`fromTarget = affordance.source === target`) and used it as a filter;
it simply never carried it onto the entry.

⚠ **A subject-afforded set of size zero renders nothing**, and for an
ordinary object with no `commandContributions` that is most of the time.
That is correct per *a section that does not apply is absent, not
hatched* — but it is worth saying plainly, because "the row is usually
missing" and "the row does not work" look identical from outside. A
noticeboard's `read`, an NPC's `talk` and a door's `open` are what it is
for.

⚠ The **radial** still shows both sources, deliberately: a radial
answers *what can I do here*, which is the wider question the gesture
asks.

## `PerceptibleMixin.getPrimaryKeyword()` surface

```ts
getPrimaryKeyword(): string | undefined;
setPrimaryKeyword(value: string | undefined): void;
```

A persistent `primaryKeyword` field on `PerceptibleMixin`
(added to a `persistent` entry in `fieldMeta`). The **primary keyword** is
the *guaranteed-resolvable handle* an MML affordance can click —
`look <primaryKeyword>` is the canonical disambiguator.

**Default behavior**: when unset, `getPrimaryKeyword()` returns
`keywords[0]` (the first entry in the derived pool — typically
the first authored keyword, falling back to the first tokenized
name word). When the authored value is set and validly in the
pool, the getter returns it. When the pool is empty (no name,
no authored keywords) and no authored override is set, returns
`undefined`.

**Fail-soft validation on the setter**: the value must appear in
the derived `keywords` pool. Invalid values are ignored with a
warning (so authors can iterate keyword sets without crashes);
state is not corrupted. The getter never calls the setter — they
are strictly independent (a stale invalid stored value is
silently shadowed by the derived-pool head).

**Set-fires-field-change**: real changes route through
`MqlSubscriptionApi.fireFieldChange(this, 'primaryKeyword', ...)`
so subscriptions on `'primaryKeyword'` (any ref record on a
Perceptible host) wake.

## `REF_FIELDS` extension: `primaryKeyword`

```ts
export const REF_FIELDS: FieldSet = [
  'displayName',
  'quantity',
  'primaryKeyword',
];
```

Every ref record shipped by the substrate carries `primaryKeyword`
for Perceptible hosts. Non-Perceptible hosts return `undefined`
from the descriptor and the substrate omits the field on the wire
(same as `quantity` for non-Globbable hosts).

The descriptor lives on `PerceptibleMixin.subscribableFields` —
contributed by the mixin that owns the gate, per the rule
`Stuff.subscribableFields` documents. Non-Perceptible hosts
contribute no descriptor; the substrate's projection loop tolerates
the absence and the field is naturally omitted from those hosts'
wire records (same shape `quantity` uses on Globbable). `REF_FIELDS`
can list `'primaryKeyword'` unconditionally because the loop's
`if (!d || !d.read) continue;` skip handles missing descriptors.

`dependsOnFields: ['primaryKeyword', 'name', 'shortDescription']`
— the getter result changes when any of the derived-pool inputs
change. `changes: [{ on: ShadowChangedEvent, by: 'target' }]` —
keyword pool can be reshaped by shadows.

## `DETAIL_FIELDS` extension: `contents`

```ts
export const DETAIL_FIELDS: FieldSet = [
  'displayName',
  'quantity',
  'primaryKeyword',
  'shortDescription',
  'longDescription',
  'details',
  'bulkMaterial',
  'mass',
  'contents',
];
```

> ⚠ **`details` is per-viewer too, and was the widest identification
> leak in the codebase.** Its descriptor read `(stuff)` while its
> neighbour `longDescription` read `(stuff, viewer)` — so the card
> enumerated every detail key *and* description regardless of what the
> reader knew, and a detail key names the part by what it does
> (`sigil`, `scorch`). `look` at least had to be asked a question; the
> card just handed it over. Both layers (`read` and `perDetailRead`)
> now pass the viewer they were always given, through to
> `Detailed.detailRoot(viewer)`. See
> [magic-items.md](./magic-items.md) § *The parts leak harder than the
> prose*.

The `contents` descriptor lives on `ContainerMixin.subscribableFields`.
For container hosts, it ships an array of `'ref'`-shape records
(via `projectFields(child, REF_FIELDS, viewer)`) for visible
contained Stuff. Non-container hosts return `undefined` from the
descriptor and the substrate omits the field on the wire.

**Per-viewer visibility filter**: the descriptor walks
`host.getContents()` and excludes anything the viewer's `Sensor`
perception check rejects (sensory occlusion, etc.). Adornments
and the actor (self) are excluded. The viewer is the
subscription holder threaded through the substrate's
`#projectStuff` pass.

**Containment add/remove diffs** ride
`FieldChangedEvent { field: 'contents' }` fires installed
inline on `ContainerMixin.addContainable` and `removeContainable`.
No new event class, no specialized add/remove diff shape — when
the field-change fires, the substrate re-projects the host, the
new `contents` array goes through the diff machinery, and the
client receives an `op: update` change carrying the patched
list. The cycle is end-to-end via the existing primitives.

The choice to put `contents` directly into `DETAIL_FIELDS`
(rather than a new `'detail-with-contents'` alias or a secondary
subscription on `things in $focus`) trades a minor inefficiency
on non-container detail subscriptions for a uniform projection
policy. If contents grow heavy enough that this matters, that's
the moment to split.

## Body discipline: percepts, not state dump

The card body renders **what a perception verb would reveal to
this viewer**, not the focused thing's internal state. `look` is
vision — it reveals appearance and gross features; an estimate at
best for hidden quantities ("looks warm," never "37.4°C").
Internal properties are not perceivable just because they exist.

This is the inspection-card reconciliation principle (see the
inspection-card slate's *Reconciliation note* and the
message-rendering-slate):
every fact has a *revelation condition* — which modality /
instrument / skill reveals it, at what fidelity. The viewer
perceives only the facts whose condition they satisfy. The card's
v1 surface walks this back to the simplest cut:

- **Player body = percept projection.** The substrate's `'detail'`
  field-set ships only percept-shaped fields (display name, short
  / long description, visible contents — already per-viewer
  filtered server-side in `ContainerMixin.contents`); the renderer
  shows them as the look output. No slot maps, mixin lists, raw
  fields, or property bags surface here.
- **Raw internal state is server-side; the v1 card has no admin
  surface.** Template path, stuff id, mixin composition, raw JSON
  dump, and `clone` / `reload` / `eval` quick actions all belong
  in a future admin surface — but the substrate doesn't project
  those fields today and no client `isAdmin` flag exists. Until
  both ship, the card carries no admin block; what authors can do
  is use the typed-command interface (`clone <template>`,
  `reload <template>`) just like any verb.
- **Per-fact revelation gating beyond visible is parked.**
  The sense/modality system (feel/smell/listen as separate
  channels), the magic lens, skill-deepens-perception, and per-
  fact provenance all wait for the perception subsystem; the
  spine (fact → revelation condition) is recorded here for the
  future build, the implementation cut is "visible" only.

### Accumulate vs. latest — v1 ships latest-only

When a viewer performs successive perception acts on the same
focus (look, then measure, then appraise), does the card show
the *union* of percepts each act has revealed, or just the
*latest* act's output?

**Choice: latest-only.** The card's `records` snapshot is replaced by
each resolve (a `card-touched` re-resolve on a static card, a delta on
a live one); there is no per-fact union across multiple `look` /
`examine` / `measure` invocations. The latest-only path stays internally consistent
because the substrate re-projects the *currently-perceivable*
field set on every re-resolve — what the card shows is what's
true *now*, from this viewer, by the modalities currently in
play.

Accumulate-per-focus is the natural target once the revelation-
condition spine lands; that work is parked alongside the
sense/modality system. v1 does not block on it; the simpler
shape ships and stays correct for the percepts the substrate
currently projects.

## Cardinality-polymorphic body

Same field-set (`'detail'` always) for the `mql` rows. The renderer
branches on result-array length:

- **Single (length 1)**: detail view — display name + long
  description (rendered via `MmlRenderer` so embedded MML
  affordances become clickable) + contents list of clickable
  affordances when the focused host is a container.
- **Multi (length > 1)**: list view — one row per match, each
  row's display name rendered as an `<item>`-affordance.

The substrate doesn't know about this branching. Cardinality-
adaptive projection (ship `'ref'` for multi, `'detail'` for single) was
considered and deferred — `'detail'` always, in exchange for a single
uniform projection policy. The minor inefficiency is accepted.

⚠ A `payload` or `client` card does not branch at all: its body is a
`CardPayload` or the client's own surface, and the length of a records
array it does not have says nothing about it.

## `find` verb

```yaml
verbs: [find]
controller: FindController
args:
  - name: query
    type: objects
    required: true
    greedy: true
    scope: ["$focus", "reachable"]
```

**Snapshot semantics.** `find` resolves the query through the
existing MQL pipeline, ships an MML list to the terminal scroll,
one row per match. No `updates_focus` (the absence is load-
bearing — defaults to `'none'`); the giver's focus is unchanged
after `find`. No subscription is registered; no live updates.

**Admin gating.** For admin / Author viewers (checked via
`MixinApi.isAuthor(commandGiver)`), each row appends the
template path in parens — `brass thermometer (/platform/thing/Thermometer)`.
Non-admins see display name only.

**Discovery.** Contributed to `PerceiverMixin.commandContributions.self`
alongside `look`, `scry`, `locate` — `find` is the
enumeration counterpart to `focus`, both surfaced on the
perceiver's verb set.

**`mql-query` integration.** The player-typed `find` rides the
command bus exactly like any other verb (controller renders
prose, player reads it). The `mql-query` one-shot wire surface
exists in parallel for future programmatic consumers — a widget
issuing a `find`-shape read without going through the command
bus. v1 does not exercise that path; the substrate is in place
for when it does.

## `mql-query` one-shot channel

Wire shape in `@saxonberg/types`:

```ts
interface MqlQueryMessage {
  type: 'mql-query';
  queryId: string;
  query: string;
  cardinality: 'one' | 'many';
  fields?: string[] | 'ref' | 'detail';
  detailKey?: string;
}

interface MqlQueryResultEnvelope {
  type: 'mql-query-result';
  frameId: number;
  queryId: string;
  result: (StuffRefRecord | StuffDetailRecord | StuffDetailFocusRecord)[];
}

interface MqlQueryErrorEnvelope {
  type: 'mql-query-error';
  frameId: number;
  queryId: string;
  reason: MqlSubscriptionErrorReason;
  detail?: string;
}
```

`MqlSubscriptionApi.handleQuery(req: QueryRequest)`:

- Reuses ONLY the parse + resolve + project pipeline.
- NO registration in `#registry`, NO dependency-index entries,
  NO listener installation. This is the "share the pipeline,
  skip the state" pattern.
- Holder and cardinality checks mirror `handleSubscribe` so a
  client's error-handling code can branch by `reason` uniformly
  across subscribes and queries.
- `focusDependent` / `locationDependent` are not carried on the
  query shape (no subscription state to wake).

`Application.processUserMessage` routes inbound `'mql-query'`
messages through `handleQuery` — same shape as the existing
`'mql-subscribe'` route. Server-side programmatic one-shot
reads call `MqlApi.resolveOne` / `resolveMany` + `projectFields`
directly; this surface is the wire-facing channel.

## `look --peek` and the phase-effects substrate

`look.yaml`'s `peek` boolean option declares a lifecycle effect
against the dispatcher:

```yaml
args:
  - name: target
    type: object
    scope: ["$focus", "reachable"]
    updates_focus: extend
    prepositions: [at]
    default: "$focus"
options:
  peek:
    type: boolean
    description: "Render prose without changing focus"
    effects:
      - { phase: focus-update, action: skip }
```

The dispatcher recognizes a small vocabulary of *lifecycle
phases* — points between parse and emit where an option can
attach a `skip` or `replace` action. The vocabulary is declared
in `api/command.ts` (`COMMAND_PHASES`, `HOOKABLE_PHASES`,
`REPLACE_HANDLERS`, `IMPLEMENTED_REPLACE_HANDLERS`, `PhaseEffect`,
`validatePhaseEffect`, `collectPhaseEffects`, `consumePhaseEffects`).
Today `focus-update` is the only hookable phase; `validate`,
`confirm-prompt`, `dispatch`, and `emit-scene` are documented
placeholders that the schema accepts but the dispatcher throws
against until their substrate lands.

When the dispatcher's positional-arg loop hits the focus-update
site, it consults `consumePhaseEffects('focus-update', model,
optionDefs)`. If any active option declares
`{phase: 'focus-update', action: 'skip'}` and is truthy on the
bound model, the focus-update step is bypassed. Pronoun memory
still updates; only the focus chain push is held back.

`LookController.execute` is unchanged — it renders the prose
body and emits the existing Scene the same way it always has.
The "peek doesn't change focus" semantic is the dispatcher's
job, not the controller's.

The generalization (a phase taxonomy plus an option-side
`effects:` declaration) lets future flags reuse the substrate
without inventing new schema fields:

- `--async` → `{phase: 'dispatch', action: 'replace', with: 'deferred-dispatch'}`
- `--explain` / `--dry-run` → `{phase: 'dispatch', action: 'replace', with: 'explain-plan'}`
- `--force` → `{phase: 'confirm-prompt', action: 'skip'}`
- `--quiet` → `{phase: 'emit-scene', action: 'skip'}`

Each lands by making its target phase hookable, not by adding a
new YAML field. See `docs/subsystems/command-routing.md` for the
dispatcher details.

## Client stuff registry

A single Zustand slice on `useStore`:

```ts
interface StuffMetadata {
  stuffId: string;
  displayName: string;
  primaryKeyword?: string;
}

interface StuffRegistrySlice {
  stuffRegistry: Map<string, StuffMetadata>;
  upsertStuffMetadata: (records: StuffMetadata[]) => void;
}
```

**Populated** by every subscription consumer the client owns.
When the wire client receives an `mql-subscription-result` or
`mql-subscription-delta` envelope, it walks every record and
calls `useStore.getState().upsertStuffMetadata([...records])`
before dispatching to widget handlers. Nested ref-shape fields
(currently just `contents`; future `equipped`, `inventory`,
etc.) are recursively walked so the registry picks up every
stuff-id mentioned anywhere in the subscription payload.

**Merge semantics**: fields present in the new record
overwrite; fields absent leave existing values intact. A ref-
only delta does not clobber detail data; a detail delta
upgrades a previously ref-only entry.

**Read by**: `MmlRenderer.commandFor()` at click-resolution
time. The renderer reads the registry directly from the global
store snapshot (`useStore.getState().stuffRegistry.get(stuffId)`)
— not via a React subscription, since the renderer just needs
the snapshot at render time. Re-renders happen naturally when
the parent (terminal, card body) re-renders.

**No eviction in v1.** Sessions are bounded; the registry is
cheap memory. The "forgotten ref leads to broken click" failure
mode (which eviction would create) is more painful than the
"registry grows unbounded" trade-off (which it prevents).

**The registry is a side-effect cache for rendering metadata,
not a source of truth for client logic.** Widgets that need live
state subscribe for it; they do not query the registry. The
registry's only legitimate reader is rendering paths that need
a per-stuff lookup (currently: `MmlRenderer.commandFor`). If a
widget reaches for the registry to answer "what's in this
container" or "is this thing visible," that's the on-ramp to a
client-side shadow model — stop and have that conversation
explicitly rather than slipping into it by accumulation.

## MML identity-tag rendering

`MmlRenderer.commandFor()` extends to four identity tags:

```ts
case 'item':
case 'name':
case 'location':
case 'object': {
  const stuffId = node.attrs['stuff-id'];
  if (stuffId) {
    const meta = useStore.getState().stuffRegistry.get(stuffId);
    if (meta?.primaryKeyword) return `look ${meta.primaryKeyword}`;
  }
  return `look ${node.label}`;  // label fallback
}
```

**Click target** is `look <primaryKeyword>` when the registry
hits, falling back to `look <node.label>` when the registry
misses or the stuff has no primary keyword. Pedagogically this
matters: `look thermometer` is what the player would type.
Showing them that command on hover and sending it on click
teaches the typed-command surface; sending an opaque `stuff-id`
would not.

**Hover preview** rides the existing `onCommandPreview` surface
unchanged.

`<direction>` and `<speech>` remain non-actionable (no
`commandFor` branch). `<exit>` continues to emit `go <dir>`.

**MML identity tags carry `stuff-id` only.** Per the architectural
rule this build establishes, do not add per-stuff attributes to
`<item>` / `<name>` / `<location>` / `<object>` beyond
`stuff-id`. New per-stuff metadata is a projection field on a
subscription, not a tag attribute.

## Shared UI components and theme tokens

The card composes from a small **shared** primitive set under
`packages/client/src/components/ui/`, not card-private styled
divs. Three rules govern it:

1. **Reusable primitives, not bespoke JSX.** Future cockpit
   widgets (inventory, self-state strip, group windows) compose
   from the same surface — and so does the layout-MML library
   the message-rendering-slate
   schedules for its Wave 2. When that lands, its `<table>` /
   `<list>` / `<field>` tags map onto these same React
   components, so the subscription-driven rendering path and the
   message-rendered path converge on one DOM shape. The card
   does not block on that library; it ships its own
   subscription-record → React-component path today.

2. **Semantic DOM = the flatten-linear-labeled floor.** Every
   primitive renders the real HTML element: `<List>` is a `<ul>`
   / `<ol>`, `<EntityName>` is a `<button>`, `<Button>` is a
   `<button>`. No ARIA props are needed to fake what the
   platform already announces. Visual-only `<div>` grids are
   the smell to avoid.

3. **Theme tokens; no hex literals.** All color / spacing / font
   values come from `tokens.ts` — semantic names (`surface`,
   `fg`, `accent`, `border`, `sectionLabel`) that a theme can
   swap wholesale. There is **no** `<color>` or `<size>` MML
   tag; coloring is a stylesheet rule keyed off semantic markup
   (the principle from the message-rendering slate).

| Primitive | Role | Renders |
|---|---|---|
| `<List>` / `<ListItem>` | semantic sequence | `<ul>` / `<ol>` + `<li>` |
| `<EntityName>` | clickable name carrying `stuff-id` | `<button data-stuff-id="...">` |
| `<Button>` | action target with `primary` / `action` / `ghost` variants | `<button>` |
| `tokens` | semantic theme values (color / space / font / radius) | `as const` exports |

### `stuff-id` is double duty: interactivity and styling

`<EntityName>` emits a `data-stuff-id` attribute on the rendered
button. The same attribute drives **two** layers from one source:

- **Interactivity** — the click target resolution layer maps
  `stuffId` (via the stuff registry's `primaryKeyword`) to the
  command this affordance sends. `MmlRenderer.commandFor`
  applies the same registry-then-label fallback for identity
  tags; the card mirrors it for contents-list rows and multi-
  focus rows via the parent's `onSendCommand` sink.
- **Styling** — a future theme stylesheet selects on
  `[data-stuff-id]` against the viewer's social-graph bucket
  (friend / foe / self) to colour the name. The
  social-graph slate +
  message-rendering slate
  describe the bucket model. The attribute is emitted today;
  bucket selectors land when that subsystem does, without any
  card changes.

One attribute, two duties — that's the slate's economy. **There
is no `<color>` or `<size>` MML tag**, and no per-tag color
attribute; coloring is always a stylesheet rule keyed off
semantic markup.

### Multi-focus rows: groups, eventually

Per [grouping.md](./grouping.md), the multi-
cardinality `$focus` result is in principle a **group** — `focus
friends` resolves a group via `GroupApi`; the card renders its
members. v1 has neither `GroupApi` nor friend/foe bucketing, so
the row shape is just "a list of styled names" — and that's the
shape it stays. When `GroupApi` lands, the card resolves the
group server-side via the same `$focus` subscription and the row
component (`<EntityName>` already carrying `stuff-id`) absorbs
the bucket selector without further work.

## Reconnect behavior

⭐⭐ **Cards do not replay, because the client never opened them.**

On `connection-established` the wire client replays the one
subscription it owns — `chrome: 'self'`, the widget shelf — and nothing
else. The CARD set is server state on the `CardRegistry`, keyed by
`Interactive`; a reconnect that lands on the same Interactive still has
its cards, and one that does not is a new session whose arrangement
`Avatar.enter` applies.

⚠ That is a real simplification over the predecessor, which replayed
every subscription's stored spec and had to reconcile the results
against a client-side card set. The failure it removes is the one that
was found live: a result arriving after the client's own unsubscribe —
which React's double-mount produces on **every dev page load** — looked
like an unknown handle and was adopted as a spurious card.

⚠ A `card-opened` for an instance the client already knows is an
idempotent overwrite, so a duplicated push cannot fork the set.

## What ships unbuilt

⚠ **Rewritten against the tree.** The predecessor's list still named
*mobile responsiveness* and *a tab strip* as unbuilt; both had shipped.
A stale hatch is worse than no hatch — it tells a reader the surface
does not exist when it does, and the wiki-search hatch in this same
build was written from exactly that kind of stale table.

- **A card the server pushes for a reason other than a command.** The
  arrangement resolver and the prompt substrate are the only two
  non-command pushes, and both are server code by construction. An
  authored or pack-shipped card would be a third tier and wants a
  resolution order across all three — a design conversation, not a
  map edit.
- **A second LIVE row.** `subject` is the only one, and adding another
  means either an existing wake or building one. `who` is the worked
  example of why: a presence wake means every login poking every open
  `who`, which is real cost for a list one click refreshes.
- **A card the player can create.** The catalogue is code; a player's
  *arrangement* (which cards, in what order) is clientState. There is
  no `cockpit card open <query>` and there must not be — the client
  supplies an identity, never a query.
- **Multi-card / split view inside one card.** One card, one body.
- **Persistent pins across reconnects.** A pin is a statement about a
  MOMENT (*keep the card about Bob open even though Bob walked out*);
  an ARRANGEMENT is a statement about a workspace and is durable. The
  asymmetry is deliberate and is worth restating because saved
  arrangements sit right beside it.
- **Animated card transitions.**
- **Per-fact revelation gating beyond visible.** Each property is
  either in the detail field-set or not surfaced; per-fact provenance
  (which act revealed this, at what fidelity) ships with the
  perception subsystem.
- **Accumulate-per-card body.** Latest-only — each resolve replaces the
  snapshot. The union of percepts across `look` / `measure` / `appraise`
  waits for the revelation-condition spine.
- **Display-flag vocabulary on `find`** (`--bare`, `--with vitals`),
  **`find --focus`**, and **pinned `find` results as a card.** `find`
  renders to the terminal.
- **Shift-click alternative on multi-row.** Plain click sends
  `look <that>`.
- **Right-click context menus on MML affordances.**
- **An admin / author projection on a card** — no `templatePath` /
  `mixins` / `containerPath` on the wire. When admin needs arrive they
  land with verified substrate rather than client-side speculation.
- **Channel stylesheets, `<color>` / `<size>` / heavy layout tags.**
  The core stays semantic.

## ⭐⭐ What the LIVE DRIVE found that the suite could not

Four defects, driven at 1440×900 and 390×844. None was visible to any
test, and two share the shape this program keeps hitting: **a rule
implemented in one of two paths silently does nothing in the other, and
a test on the path that has it passes.**

1. ⚠⚠ **Two cards of one kind rendered the SAME pin command.** `who`
   and `who --here` are different commands, so different cards — but
   both controls read `cockpit card pin who`, and the server resolved
   the catalogue name first and acted on whichever it found. **Two
   controls, identical labels, acting on one thing.** Fixed by making
   the card's ref its KEY (quoted when it contains whitespace, or
   `--here` binds as an option of `cockpit`), and by resolving the key
   first server-side. *A control that does not act on the card it is on
   is worse than a missing one.*
2. ⚠⚠ **The phone ignored both feed filters.** `shell.result` and the
   named-view filter lived inside `CardFeed` — the DESKTOP rail — so the
   inline stack honoured neither. The per-viewport override's *entire
   payoff* is on the phone, and the phone was the path that ignored it.
   Both paths call `visibleCards` now, and a source guard asserts they
   still do rather than re-deriving it.
3. ⚠ **The news card was unreachable by command.** `press` carried
   `requiresPublisher` at VERB level, so a player holding no publishing
   position was refused *"there is nobody you can press as"* for a bare
   `press` — a publishing answer to a reading question. The gate moved
   to the `post` / `edit` / `retract` subcommands; `PressApi.publish`'s
   own `EmploymentApi.mayPublishAs` entitlement was doing the real work
   all along. `press` also left `AuthorMixin`'s contribution list for
   the ordinary one — contributed only there, the news was a surface an
   ordinary player could not ask for at all.
4. ⚠ **`cms` and `studio` did not exist as verbs.** A command YAML that
   nothing CONTRIBUTES is not reachable, whatever the catalogue says.
   They join the author suite beside `git`.

⭐ **And three things the drive CONFIRMED** that only driving could:
the arrangement really does apply on login (a cold reload into `build`
comes up with the editor and the git panel); the live card really does
carry no timestamp and no refresh while every static card does; and the action row really does show `register terminal` on the
TPA terminal rather than `cast · defend · destruct` on everything.

### ⚠ One finding for the reader, outside this build

At 390px a long unbroken **monospace command echo** (`settings set
shell.result.mobile terminal`) forces the terminal's row past the
viewport, and the terminal's own `overflow-x: auto` scrolls the prose
out of reach. It is not the ICB trap — `documentElement.scrollWidth`
stays 390 — and it is not this build: `Terminal.tsx`, `TabStrip.tsx`
and `primitives.ts` are untouched by the branch, and a `clear` restores
a 390px terminal exactly. Any long command does it. Recorded here
because it is a real phone problem that a card-surface reader will hit
while checking these findings.

### ⚠ Recorded, not closed

- **`chat on` does not wake an open rail.** Firing from
  `SubjectCatalogue` was tried and reverted (it breaks
  persist-then-fire). The card push is the obvious new seam — `chat on`
  could open or touch a card, and the push would be the wake — but the
  rail is not a card, so this stays recorded rather than half-wired.
  **Do not re-try the reverted seam.**
- **`HERE` rows render `something` — and the requirements' framing of
  WHY is wrong.** They say two visibility gates disagree:
  `Container.contents` keeps a child on `PerceptionApi.perceives`, then
  `projectFields` re-points `displayName` through
  `describeFor`, whose `canSeeGate` says no.

  ⭐ What the tree shows is that they answer **different questions**.
  `perceives` is the **concealment** gate (*is it hidden from you*);
  `canSeeGate` is the **light** gate (`VisionModality.canSee` → the
  perceived band against `REQUIRED_BAND_FOR_DETAIL`). `LookController`
  applies **exactly the same pair**, so the card and the prose agree by
  construction — both would say `something` in the same conditions.

  So the likely defect is not gate arbitration but the **light band of
  ordinary rooms**, which is a light-model question rather than a
  perception one. Recorded here rather than fixed, because the next act
  is to read the band at the room the report came from — and if lit
  rooms also render `something`, the fix is in the light seeding and
  this framing was the thing that was wrong.
- **The radial's `stuffId` on transcript nouns.** Fourteen emitters
  interpolate `getName()` / `getPresentation()` into `Mml.compose`,
  which produces escaped TEXT rather than an identity-bearing tag, so
  the radial has nothing to hook. Most are confirmations and greetings
  where a radial would be meaningless; the handful that name a real
  world object want `Mml.thing` / `Mml.actor`. Bounded and recorded
  rather than swept, per the plan's own instruction not to open the tag
  vocabulary over it.

## Known future considerations

Flag-don't-fix; just record so future debugging knows the
substrate's choices.

### Per-viewer presentation isn't modeled on the client

Disguise / recognition will eventually want different
`displayName` / `primaryKeyword` per observer. The substrate
already projects per-viewer (each subscription's `viewer` is the
holder's Interactive, so wire payloads are already viewer-
specific). The single client-side `stuffRegistry` slice assumes
one viewer per session, which is true for v1's player-only
client; admin spectating, multi-viewer surfaces, or any "see
this through another's eyes" feature will need to revisit the
registry's keying. Likely shape: a `viewerId` axis on the Map
key, or a per-viewer sub-slice that the renderer routes through.

### Last-writer-wins on conflicting records

Two subscriptions could ship records with different `displayName`
for the same stuff-id — legitimately (under per-viewer
projection, recognition state shifting between subscribes) or as
a bug. Today's merge ("fields present overwrite; absent stay")
accepts the first case silently and provides no signal for the
second. Worth knowing when debugging "why does this thing's
name suddenly differ from what I expected." A future contention-
detection pass could log when an upsert overwrites a non-empty
field with a different value; today the registry is intentionally
quiet.

### Eviction policy upgrade path

Sessions are bounded today, so the never-evict policy is fine.
When sessions get long (long-running NPC tutors, persistent
classroom sessions, etc.), eviction will need a strategy (LRU,
reference-counted by active subscriptions, or session-cap). The
upgrade is clean: consumers only call `Map.get`, so any eviction
policy lands behind the existing read shape without changing
call sites. The decision point is when the registry footprint
crosses whatever profiling threshold makes it the next
optimization candidate.

## Build history

Several substrate shapes shifted during MR iteration. Recording
them here so future debugging knows the substrate's choices
weren't always the obvious ones:

- **Canonical-kind registry retired.** The build initially
  shipped a server-side `MqlSubscriptionApi.registerKind` /
  `CanonicalKindSpec` registry that let clients subscribe by
  name (`'me.focus'`). The registry was a pure server-side
  macro over the wire — same bytes, alias-only — so it was
  demolished in favor of clients sending the raw spec
  (`subscribeMql(spec)`). The substrate's `focusDependent` /
  `locationDependent` flags are now part of the request,
  not derived from a registered kind.
- **`me.location` subscription landed.** Added as a second
  client-issued subscription to drive the breadcrumb root
  (separate from the focused-thing body). Required adding
  `locationDependent` to the substrate's `SubscribeRequest`
  shape and firing `FieldChangedEvent { field: 'container' }`
  from `Containable.setContainer`.
- **`primaryKeyword` descriptor relocated.** Originally on
  `Stuff.subscribableFields` with an inline mixin gate
  (`MixinApi.isPerceptible`); moved onto
  `PerceptibleMixin.subscribableFields` per the rule "mixin-
  gated renders go on the mixin that owns the gate."
- **`getMarkupLong` relocated, augmenter pipeline added.**
  Originally on `Detailed` interface/impl with `VisibleMixin`
  duck-typing into it. The method moved to `VisibleMixin` and
  the wrap-detail-keys logic became the first
  `MarkupAugmenter` contribution. Substrate added: the
  `MarkupAugmenter` type + `augmentMarkup` helper in
  `api/mml.ts` and the `MixinApi.getAllMarkupAugmenters`
  prototype-chain walker. Future contributors (exit-direction
  auto-link, language gating, spoiler hide) plug in via
  `static markupAugmenters` on their mixin.
- **`skip_focus_when_option` retired in favor of phase
  effects.** Originally a single-purpose YAML field on
  positional args; replaced by the dispatcher's phase /
  effects vocabulary in `api/command.ts`. The `look --peek`
  YAML now declares
  `effects: [{phase: 'focus-update', action: 'skip'}]`.
- **Unified breadcrumb.** Two parallel strips (top focus
  breadcrumb + in-body detail trail) collapsed into a single
  strip: root + trail + detail segments. The in-body
  `DetailTrail` was deleted.
- **First-delivery auto-paint.** The original spec said "focus
  changes clear the body; explicit `look` paints." On a fresh
  session that left the card sitting on the placeholder until
  the player typed `look`. First-delivery auto-paint elides
  the cold-start step without changing the focus-shift-clears
  rule.
- **Admin extras removed.** Originally shipped as forward-
  compatible scaffolding (template-path + mixins + container-
  path display, `clone` / `reload` / `eval` buttons). Removed
  because the substrate doesn't project the fields and the
  auth slice has no `isAdmin` flag — the scaffolding was
  permanent dead code. Future admin needs land with verified
  substrate.

Commit range: `41240c7..HEAD` on the `inspection` branch.

### The card-surface build (`build/card-surface`)

The wave that turned the inspection *pane* into the card *surface*. What
it retired is the majority of it:

- **The focus signal.** `usePaneFeed`'s focus-watching effect,
  `openSubjectPane`, the `inspect` and `location` catalogue rows, and
  the App-level paint/clear policy with its client-side verb peek. The
  lesson paint/clear taught — *focus is a pointer; look is the verb that
  paints* — is now taught by `look` minting a card, which is stronger.
- **The five holds** (`CardHold`, `HOLD_WAKES_ON`, `evaluateHold`,
  `anySubject`, `emitReleased`, `MqlSubscriptionReleasedEnvelope` and
  its four spatial reasons), for one relevance window plus `pinned`.
- **The switcher** (`PaneSwitch` / `PaneTab` / `PaneSlot`, `rightPane`),
  and with it `WhoPane` / `NewsTickerPane` / `WikiPane` as hand-written
  surfaces with their own data paths. Their row-rendering knowledge was
  **salvaged** into `CardBodies`; what died is the pane shell, its 360px
  chrome and its tab.
- **The CMS's own four-tab mode bar** (`CmsSurface`) — a second
  switcher, in the second column, for the same expired reason.
- **The client's husk `setInterval`**, so there is exactly one clock.
- **`pushed: true`**, because the distinction it drew disappeared: the
  client never opens a card, so every `card-opened` is a push.

Two things were changed AGAINST the plan and are recorded where they
live: the inspection card ships **live** (the plan's table said static
while its driving script drove "the one live card"), and the
`shell.result` filter keys on **`meta.carded`** rather than on the topic
`shell.result` (the per-card prose audit falsified decision 10's
premise).

### ⭐⭐ The second half of the same branch (`fda6aeacc..HEAD`)

The substrate above was driven and the verdict was *"we're clearly not
on the same page as far as the experience we're trying to create."* The
model was wrong in **shape**, not merely buggy, and the correction is
the more important half of the branch. Twelve commits, and everything
above already reads as the corrected model — this note exists so a
reader who remembers the first version knows what moved and why.

What the model is, stated once:

- The **terminal is the ground-level record of proof** that things
  happened. Filterable; several views over segments of the record.
- **Cards are structured views into the same events** — where tables
  and interactive forms belong. Also a feed, also carvable by filter.
- The whole thing is about **managing attention**.

Four things changed, each of which had a symptom nobody could see from
a green suite:

1. **The feed became a LOG.** Dedup-on-command meant asking twice
   changed nothing visible, and the suppressed prose made it look like
   the command had done nothing at all. Only `singleton` rows reuse now.
2. **Liveness moved onto attention.** `demoteLive` — newest card holds
   the subscription, everything behind it is a snapshot.
3. **Inspection collapsed to ONE row.** `place` folded into `subject`;
   the four things you can look at are `StuffKind`s of the subject.
4. **`meta.carded` became a fact.** It carries the card's `instanceId`,
   stamped only after an open returned one.

Plus the substrate fix all of it stands on: **subject-bound
subscriptions**, perception-gated on every re-resolve.

⚠ **What ships unbuilt, restated for this half:** no tables, no forms,
no interactive cards — the widest gap between the stated vision and what
ships, and additive on top of this substrate. Fixed tabs rather than
tags. And on the self card, actor-afforded verbs render as the entire
verb list, which wants the `+N more` cap the composition row already
has.
