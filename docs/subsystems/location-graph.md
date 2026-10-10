# The location graph — the world's shape, and what a player knows of it

Two artifacts that **never join**, and the gap between them is the
design:

| | the graph | a player's map |
|---|---|---|
| what | every place in the realm and every exit out of it | the places one player has been, and the edges they used |
| where | the `location_graph` collection | a document at `/home/<key>/map/<locality>` |
| truth | **derived** from the content rows; droppable | **a claim**, written when perceived, never corrected |
| who reads it | the lints, the publish gate, the router, the survey act | that player |
| reaches a client | ⛔ **never** | yes — it is theirs |

⭐⭐ **The evidence firewall is structural, not policed.** A player's map
is not a filter over the graph; it is a separate artifact written from
what they perceived. Nothing joins them, so no read can hand somebody
the shape of places they have not earned. The map writer does not
import `PlaceNode`.

---

## The durable per-instance handle

Everything here keys on a place by its **durable handle**
(`Stuff.getDurableHandle()`), never by its template path. One row backs
forty provisioned dorm rooms, so lineage is the wrong key for *this
room*; `stuffId` is fresh per construction, so it is the wrong key for
*this room after a reload*. The handle is the rule **an identity exists
iff the NAME is durable** — re-derivable or recorded — expressed as a
string. Three rungs, on the hosts that own the facts:

| rung | host | answer |
|---|---|---|
| minted | `Stuff` | the stamped identity, when it differs from the row |
| keyed | `PersistableMixin` | `` `<row>#<key>` `` once the key is **explicit** |
| singleton | `SingletonMixin` | the row — the one instance IS it |

Full contract, including why the `#` joiner is load-bearing and why an
ephemeral clone must answer `null`:
[location.md § The durable per-instance handle](./location.md).

`Exit.getDiscoveryKey()` is built on it
(`<source-handle>#exit:<dir>`), which is what stops a secret found in
one dorm room reading as found in all forty —
[concealment.md](./concealment.md).

---

## What is a node, and what is not

⭐ **A node is a place, and *one row is one place*.**

- A **template node** (`origin: 'template'`) is a content row whose
  effective class extends a Location root **and composes
  `SingletonMixin`**. Its identity IS its row.
- A **plan node** (`origin: 'plan'`) is a circulation node a warren's
  `PlatPlan` declares — a corridor, a road segment. Its identity is
  `PlatPlan.nodeIdentityOf(nodeId, parentExtent)`, which is
  re-derivable from the plan plus the extent, so it survives the node
  being reaped and re-minted.

⚠ **Everything else is a KIND**, minted many times through a warren or
a programme, and is deliberately **absent**. A holding's interior is
somebody's house behind a gate; the gate is all the graph knows, as one
**slot stub** per frontage. The elastic half of the world is perceived
live and never stored — which is also why a lounge satellite is nothing
here, and nothing on anybody's map.

⚠ **Place rows are not all under a `/location/` segment.** Several
predate the `<root>/<branch>/` path pattern, so enumeration is **by
class**, always — never by path infix.

---

## The invariants

`lib/location/GraphInvariants.ts` is one instance value class over
plain data, run by **two** callers: `check-location-graph` over the rows
on disk, and `NavigationApi.checkGraph` over the projected nodes at
runtime. Two copies would drift, and a gate and a runtime check
disagreeing about what a dangling exit is reads as neither of them being
wrong.

**Three errors** — the world is broken or will break:

| rule | what it catches |
|---|---|
| `dangling-destination` | a destination naming nothing. **Today a boot crash** |
| `bidirectional-both-sides` | the same pair declared `bidirectional` twice — the engine installs the pair twice |
| `published-into-unpublished` | a live room pointing into draft content |

⭐ The publish rule runs **one way only**: `published → unpublished` is a
dangling edge waiting to happen, while `unpublished → published` is how
a draft zone attaches to the live world when it lands.

⚠⚠ **`bidirectional: true` means INSTALL BOTH SIDES.** Such an edge is
reciprocal by construction and the far side must not declare it again —
`Exitable._applyExitSpec` calls `addBidirectionalExit`. A rule written
as *"a bidirectional edge the far side does not reciprocate"* flagged
Duncan Hall's correctly-authored front doors; that phrasing has no
referent in the real content format.

**Four questions** — censused and ratcheted, because a one-way passage
is legitimate content and an author must be able to ship one:
`cross-zone-one-sided`, `unreachable-from-entrance`, `asymmetric-edge`,
`destination-is-a-kind`.

⚠⚠ `unreachable-from-entrance` has **false positives by
construction**: code-installed exits are not in rows (a warren's hub
exits, `DormDoor`, `FloorStairExit`), so a zone reached only through a
code-installed door reads as entrance-less. It earns its place because
the count may not **grow**. Counts and ceilings:
lint-family.md.

A zone's **entrances** are derived: the nodes with an inbound
cross-zone edge, plus every travel stop in it, plus the default start
location when it lies there. A zone with no entrance at all is **one**
finding, not one per room — reporting it forty times is how a census
becomes noise nobody reads.

---

## The collection

`location_graph`, one row per node (`PlaceNode`, a `Document` —
`ParcelRecord`'s shape; nothing composes or clones it).

⭐ **`reset: keep`, and the reasoning generalizes**: "derived" is an
argument for being *droppable*, not for being dropped nightly. The
nightly job does not restart the process, so a wipe would leave every
lint, publish gate and board read blind until the next boot rather than
for a moment. There is nothing to gain and a window of blindness to
lose.

`generation` is the rebuild's sweep key: `rebuild()` stamps every
projected node with a fresh generation and deletes the rows carrying any
other — droppable-and-rebuildable with no window in which the graph is
empty. ⚠ **Monotonic, not `Date.now()`**: a wall-clock stamp collides
when two rebuilds land in the same millisecond, and a colliding
generation sweeps *nothing*, because every stale row's stamp equals the
new one. The sweep then silently keeps nodes for rows that have stopped
being places — exactly the failure the generation exists to prevent.

⭐⭐ **The registry warms LAZILY, on the first graph read — there is no
`onCreate`.** Nothing reads the graph at boot: the traversal gate's
publish check goes through `ParcelApi`, and the invariant check, the
re-projection and the router's queries are all later events. So an eager
walk of every content row at boot would be work for no reader. The boot
manifest entry exists so the registry *exists* for `NavigationLogic` to
find; `ReadingCatalogue` is the precedent for the lazy half. The warm is
guarded, so a burst of first reads builds once.

⚠ `PlaceNode` carries **no finder statics**, unlike `ParcelRecord`. The
registry is its only consumer, so the queries are private methods there
over the inherited `Document.find`; the record itself holds only
`toGraphNode()`, a fact about the row rather than a lookup of rows.

## The write chokepoint

`DomainHook` (`AroundSave` + `AroundDelete` over `Collections.Content`)
is **the real template write chokepoint** —
`TemplateApi.saveTemplate` has five direct callers and is bypassed by
the pack installer and by `Template.save()`.

On save it re-projects the row **and every child that `extends` it**
(editing a parent changes every child's effective exits), then runs
`checkGraph(path)` and records each finding against the row through
`DiagnosticApi` — the author's own channel, on the live stream and in
`errors`.

⭐⭐ **A throw in the projection never fails the save.** It is caught,
recorded as a diagnostic against the row, and swallowed: the graph is
derived and self-heals at the next rebuild, and losing an author's save
to a hiccup in a derived store would be the wrong trade. An author who
cannot save because an index is unhappy has no way to understand why.
⚠ A *validator* throw still refuses the save — that gate is not this
one's to soften.

On delete the path is resolved **before** `next`, because afterwards
there is nothing to resolve it from and the graph would keep a node for
a row that is gone.

## A dangling exit is a diagnostic, not a boot crash

⭐⭐ `StuffApi.singleton` throws *Template not found* for a missing row,
and eager rooms come from each pack's `boot:` list — so **one mistyped
destination anywhere in content took the whole world down**, wrapped as
*"failed to clone"*, from a stack trace naming the framework rather than
the row. An author could not see it coming and could not read it when it
arrived.

Now the direction is installed **unbuilt**: `look` still names it, the
destination path is kept, `canTraverse` refuses with `gate: 'unbuilt'`
and *"Nothing lies that way yet"*, and the author is told which row and
which direction.

⭐ **It heals.** An existing unbuilt stub is *replaceable*: create the
row the author meant, re-hydrate, and the real exit lands where the stub
was. Without that the stub would win forever and the fix would look like
it had not worked. ⚠ The reverse is refused — a *working* exit is never
traded for a stub.

⚠⚠ Caught **narrowly**, by the one message `StuffApi.clone` throws for a
missing row, with everything else rethrown: a failure inside a
destination's own `onCreate` is a different fault and must stay loud.
And by a catch rather than a pre-check — a `Template.findByPath` before
every `singleton` adds a store round-trip per authored exit on the
hydration hot path (199 edges at boot) and makes exit installation
depend on a reachable store, which took six suites red on
`isConnected is not a function` and is the same hazard a live server
hits during early boot.

⚠ `Exit._unbuilt` is **transient**. It is a fact about the world as
loaded, and the next hydrate re-derives it; persisting it would outlive
the defect it describes.

## The two far-side gates

`TraversalGate` gains `'unpublished'` and `'unbuilt'`. Both run **before
the lock gate**, so a wall reads as a wall rather than as a locked door,
and both follow the lock gate's own rule: **never resolve the
destination**. One reads a transient flag; the other is
`ParcelApi.isPathPublished`, a sync longest-prefix read.

⚠ An **untitled** path reads as published. There is no parcel there to
be a wall, and `lint:untitled` already forbids shipping an untitled
path — so the honest default for *nobody has said* is *not a wall*,
rather than silently sealing ground whose title somebody forgot to
declare.

`published` is **denormalised** from the covering parcel so
`Exit.canTraverse` can read it synchronously without resolving a
destination (the rule the lock gate already follows). The parcel stays
the source of truth; a flip re-projects the extent.

---

## A player's map

The other artifact, and the one that **does** reach a client — because
it is theirs.

One document per `(player, finest covering locality)`, at
`/home/<key>/map/<locality address>`, kind `map`. ⭐ The address keeps
its slashes, so a coarser read is a **prefix read with no join**
(`map terminus` is every map filed under Terminus) — and copying one
locality's map hands over nothing about any other, which is the only way
map-selling can ever work.

⭐⭐ **It is a CLAIM, not a view.** Nothing joins it to the graph. The
map writer does not import `PlaceNode`, so no read can hand somebody the
shape of a place they have not earned.

⚠ And the **live room is the right source** for a second reason: it
carries the elastic nodes the graph deliberately does not store (a
holding's rooms, a corridor minted on approach). A map built from the
graph could not record a dorm room at all.

### Four channels, three of which can be wrong

| channel | revealed by | can be wrong |
|---|---|---|
| `perception` | you saw it, or walked it | yes, if it changed |
| `publication` | a timetable is public | yes, if a station went dark |
| `told` | somebody said so | ⚠ yes, and they may have **lied** |
| `bought` | a transaction | yes, and that is the seller's reputation |

⚠ `told` and `bought` are **vocabulary with no writer**. They are in the
shape from the start because retrofitting provenance onto a store that
assumed one channel is the expensive version; `told`'s attribution
(somebody lied to you, and the record should say who) belongs with
[accountability.md](./accountability.md).

### ⭐⭐ The growth rule, and why rot is the feature

An observation identical in `(kind, place, dir, to, toLabel, channel)`
to the **latest** claim for that key bumps its `lastSeen`. A differing
one is **appended**. **Nothing is ever removed and nothing is ever
corrected.**

That is what lets a map be wrong. A merge would pick a winner and hide
that it did, which turns a knowledge model back into a truth model — and
*"who was there more recently"* dissolves anyway once a map is many
claims rather than one document with one date.

⭐ `channel` is part of the key on purpose: *you saw an exit east* and
*somebody told you there is one* are two different claims about the
world. ⚠ So is `toLabel`, and that one was found by a test: an exit
whose far side CHANGED is usually only distinguishable by its label,
because `to` is null for anything not resident — with `to` alone,
east→yard and east→cellar keyed the same and the second silently bumped
the first, erasing the disagreement this whole model exists to preserve.

⭐ Nothing changed ⇒ **no document write**, which is what bounds a map's
growth to the number of *distinct* observations rather than to how often
somebody types `look`.

### Two shapes of disagreement

- **The far side changed** — two claims for one direction, and both
  render with their dates.
- **The exit vanished** — and this is the common one, which records
  *nothing at all*, because the player saw no east exit to write down.
  The comparison is then against the **place's own latest observation**:
  the place claim is the *I looked here at T* record, so an edge older
  than the latest look is an edge that was not there last time anybody
  looked. ⭐ Which is why nothing has to be merged or deleted to make a
  map honest — the staleness is derivable from two timestamps the
  document already carries.

### The seams

Two optional `@hook`s on the `Perceiver` interface —
`onPerceivedPlace?` and `onReadTimetable?` — plus `Mobile.onTraversed?`.
Declaring an optional hook claims nothing of a composer that does not
implement it, so an NPC perceiver stays a no-op.

⭐⭐⭐ **And then both hooks were deleted, because neither earned its
keep.** `Perceiver.learnSurroundings(location, occupants?)` and
`learnTimetable(stops)` are what a verb calls, and they reach each
recorder **directly** through `MixinApi.isCartographer` /
`isBeliefStore` — the same shape as the two recorders that predate this
build (`BeliefStore.learnIdentityOf`, `PerceptionApi.recordDiscovery`).

⚠⚠ `onPerceivedPlace` and `onReadTimetable` had **one implementer
between them** (`CartographerMixin`), and the generality claimed for
them was a hypothetical second map-keeper — the same use case wearing a
different hat. An optional hook also cannot be invoked without a
structural cast, which is where the three controller casts came from.
⭐ **A hook earns its keep by having more than one implementer**:
`Mobile.onTraversed` does (the cartographer and `RespirationMixin`), so
it stays a hook; these did not.

⛔ **And the names were wrong.** What is recorded is **navigational** —
*I was here*, *this way leads there* — not perceptual. The method is
`learn*` to sit beside `learnIdentityOf`; *"surroundings"* is the
game's own word for the room-level read (*"Your surroundings are
indistinct."*). The claim's channels are `walked` · `seen` ·
`published`, and the two fields that dressed the record as sense data
are gone: `modality` was the hardcoded literal `'vision'` (from a local
variable misleadingly named `band`) that **nothing read**, and `band`
itself was declared and **never written**.

⚠⚠ The tell that the old vocabulary was wrong: `MapController` mapped
`channel === 'perception'` to the literal word **"walked"** — the view
layer had been translating a model that called walking a kind of
seeing, and the writer apologized for it in a comment. The translation
is deleted; the channels are the words, so a new channel needs no edit
in the renderer. ⭐ *Walked* and *seen* are now different claims that
both render — previously unrepresentable, since one channel covered
both and printed "walked" either way.

⛔ `told` and `bought` are **cut**. They were vocabulary with no
writer, kept on the argument that retrofitting provenance is expensive;
an axis with two live values and two imaginary ones teaches the wrong
shape, and `told`'s attribution belongs with the accountability ledger. The build shipped without that distinction
and the cost was immediate: `look`, `look`-in-the-dark and `sense` each
fired `onPerceivedPlace` **through its own structural cast**, each
carried its own copy of the same ten-line ordering comment, and the
`Exitable` narrowing and the perception gate were re-assembled at every
one. An `@hook` is by definition something the framework invokes, and
*"the framework"* had quietly become three command controllers.

⚠⚠ **And they had already diverged** — which is the finding, not the
tidiness. `look` also recorded WHO it saw (`learnIdentityOf`, the
repeat-perception write); `sense` never did. So the one verb an
arriving body is forced into (`autoSenseOnArrival` →
`forceCommand('sense')`, for players and NPCs alike) noticed the room
and nobody in it. Both halves of the perception moment — the place and
the people — now live in `learnSurroundings`, so a fourth verb that
describes a place gets them by calling one method.

⭐ `Mobile.traverse` is the precedent and was right all along: it fires
`onTraversed` on the mover from inside the move, through a local
optional-hook dispatcher, and no controller has ever had to know that
`CartographerMixin` exists. `Perceiver` now has the same dispatcher for
the same reason.

⭐⭐ **The implementer is `CartographerMixin`** (`lib/location/`), not
`Avatar`. It encapsulates one concern with three parts: **the policy**
(do I keep a map, and under whose key), **the conversion** (live `Stuff`
into plain claims — the durable handle, the locality address, the
grouping address), and **the routing** of the three seams into
`NavigationApi.recordPlace`.

⚠ It lived on `Avatar` first — ~220 lines of it — and the tell that it
did not belong there was a private `writesMaps()` predicate
**re-narrowing the host set from inside the class**, which is this
repo's own signal that the host is wrong. The deeper sign was in this
very doc, which said *"an NPC that wants a map implements the two
`Perceiver` hooks"*: true, and under that shape it would have had to
**reimplement the whole conversion**, because all of it was private to
`Avatar`. It composes one line now, and a test pins exactly that — a
host that is not an Avatar keeping a map.

⭐⭐ Eligibility moved to where it is answerable, and the default is
**`getDurableHandle() !== null`** — *a map-keeper needs a durable
handle for the same reason the places it records do.* A map is filed
under a name and read back later; a host with no name that outlives it
has nowhere to file one.

⚠⚠ **This closes a real pooling defect, not a theoretical one.** The
default was `true`, and `mapOwnerKey()` falls back to the template path
for a host with no minted identity — so every `Extra` cloned from one
row filed into **one shared map**. Two sentries from the same row probe
identical owner keys. An `Extra` is a *role*, not a person: no
`SingletonMixin`, no mint, no handle, and now a clean decline. A `Cast`
(one person per path) gets its row through the singleton rung and
keeps one.

⚠ **The handle is the PREDICATE, never the key.** `mapOwnerKey()` stays
`getIdentityPath()`: an Avatar that has saved once reads a compound
`` `<row>#<key>` `` handle, so keying the map on it would silently move
a player's map the first time they persisted.

⭐ It is still not the mixin going looking — it reads one fact the host
already answers about itself, rather than narrowing its own composers
by shape. `Avatar` chains through `super` and adds the one fact only
the family can answer: a wire body persists nothing (a circle's geography is
not real geography and must not be written onto the person wearing the
body) and a guest is a throwaway persona — **both already answer
`shouldPersist() → false`**, so the override reads an existing honest
fact rather than inventing a second one. Documents are `pass` under the
sandbox, so that check is the only thing stopping a circle visit from
writing real geography.

⚠⚠ **The `map` VERB stays on `Avatar`**, deliberately: *writing* a map
is a capability, *reading* one is a person's — the document is under
`/home/<key>` and NPCs do not type. Keeping them apart is what lets an
NPC guide keep a map without being handed a verb it can never use.

⭐ `learnSurroundings` runs `obviousExitsFor(viewer)` **before** it
calls any recorder — *the perception moment*. The list has already been filtered
through the perception gate, so the hook **cannot learn about an exit
the viewer could not see**. That is what makes the firewall structural
rather than policed, and it is now guaranteed by one method rather than
by three controllers each remembering the order. It also
**returns** that list, because it is the same list the verb renders and
computing it twice is how the transcript and the map could ever
disagree. It fires in the dark too: you cannot describe a pitch-black
room, but you have been there and can feel the ways out.

⚠⚠ **The writer runs inside a FORCED frame.** Arrival auto-senses via
`self.forceCommand('sense')`, and `getActingAuthor()` returns `null`
when any frame in the chain is forced — so the ordinary context gate
would reach `canAtPath(null, …)` and fail closed for the single most
important write this kind has. `DocumentApi.saveMap` is therefore a
**derived-owner writer** (the fifth ownership bypass, on
`saveInstrument`'s rails): no caller-supplied owner, the path pinned
under that player's own `/home/<key>/map/`, the `kind` pinned, and the
face gated to `NavigationLogic`.

⚠ Every writer is fire-and-forget and swallows its own errors. A map is
a convenience; failing somebody's `look` because a document would not
save is the wrong trade.

### The read

`map` (the index of localities you hold a map of) and `map <locality>`
— the places grouped by the address the content declares, each with the
ways out you know and how you know them.

⚠⚠ **That grouping had never once worked, and the routing build's
sweep found it.** `CartographerMixin.groupingAddressOf` duck-typed
`getDeclaredAddress?.()` — a method that exists **nowhere**;
`AddressableMixin`'s reader is `getAddress()`. The optional call
answered `undefined` every time, so `MapClaim.group` was never
populated and every place fell into the unnamed bucket, which renders
identically to having no groups at all. 53 of the realm's 128 places
declare an address. ⭐ It was found by routing *needing* it to narrow a
destination, and routing's own tests had passed because their fixtures
set `group` by hand — working perfectly against data the writer never
wrote. See [antipatterns.md § A duck-typed optional call on a method
name NOBODY DEFINES](../antipatterns.md).

⭐ **AC14 is structural**: the controller's single read is
`NavigationApi.readMap(viewerKey, prefix)`, which resolves under the
actor's own home and nowhere else. Nothing in it touches
`location_graph`.

⚠ *"You have no map of X"* rather than an empty map of X. An empty
rendering would read as *there is nothing there*, which is a claim about
the world; this is a claim about the player. ⚠ An ungrouped place gets
**no invented heading** — inventing one would be the map asserting a
building nobody authored.

⚠ No card. The inspection card is laid out by `StuffKind` and a map is
not a Stuff; the renderer and its card are `map-slate`'s.

## ⭐⭐⭐ Routing — one traversal, or none

The index exists so somebody can plan over it, and this is that
somebody. The whole of it is four reads on `NavigationApi`
(`routeBetween` · `routeOnMap` · `reachFrom` · `costMatrix`, plus
`routeOverEdges` for an authored lane), one value class
(`lib/location/Traversal.ts`), and a gate that makes *one traversal*
literal rather than aspirational.

### The skeleton, and what it refuses to own

`Traversal` owns the frontier, the visited set, the bounds, the
recursion order and the expansion count. ⛔ **It owns no policy.**
Hazard guards, atmosphere refusal, `published`, mode admission and cost
caps all live in the caller's `neighbours` / `descend`. A skeleton that
knew about doors would be the twelfth walk with extra steps.

Three orders: `depth-first` (the perception walks' fold-up),
`breadth-first` (a reach, mark-on-dequeue), and `cheapest-first`
(Dijkstra, ⭐ with a deterministic `keyOf` tie-break, because *"the
router picked a different road today"* is a bug report nobody can act
on). It is **synchronous** by design: the perception entry points are
sync (`signalAt` returns a `Light`) and making the walk async would
change their call shape across every consumer, so async callers
materialise a graph first and then walk it. That is also the shape
hierarchical search wants.

⚠⚠ **The depth gate fires BEFORE the visited mark**, and this is
observable: a node first reached at depth 3 is refused *without being
marked*, so it stays reachable at depth ≤ 2 by another path. Marking it
would silently darken rooms.
`scripts/__tests__/golden/perception-characterization.json` holds every
place in the realm to the pre-migration numbers — lux, dB, ppm and
every gather arrival **with its printed compass direction** — because
all four perception walks are order-dependent and that order is
preserved deliberately, not inherited by accident.

⭐ **The prior decision was narrowed, not overturned.**
`Modality.ts` recorded *"per-modality walks, not a generic walker"*,
and it was right about the accumulators (light caps all its openings as
one; sound and smell attenuate per child; smell's dominant identity
turns on walk order; the gather pushes a level down and emits
pre-order) and wrong about the frontier, which was identical in all
four. **One skeleton, four accumulators.**

### Bounds, and why only one of them is a budget

`{ hops?, nodes?, cost? }`. `hops` and `cost` are **natural limits** —
the walk simply does not expand past them. `nodes` is the **budget**:
reaching it is a *refusal*, reported as `exhausted: 'nodes'` and turned
into `{ ok: false, reason: 'budget' }`.

⭐⭐ Conflating the two is how a caller comes to believe *"there is no
way"* when the truth was *"I stopped looking"* — a lie the caller
cannot detect. Every search takes a **caller-declared budget with no
default**, and every refusal carries `expanded`, which is both the
evidence and the input a compute meter would need.

⚠ A budget is a **performance** bound and never a knowledge one. What
a character knows of the world is 100% its author's to declare
(`knows:` in brain config, `extent` on the knowledge source); what it
may think about in one beat is the operator's
(`navigation.attendedSearchBudget`, 400). Conflating *those* would be
the engine deciding how much of the realm a character has heard of.

### Two knowledge sources, and the firewall between them

```ts
routeBetween(from, to, profile, { extent? }, budget)   // the WORLD index
routeOnMap(viewerKey, prefix, from, to, profile, budget) // one person's CLAIMS
```

The world source is omniscient: legitimate for a brain whose author
declared what it knows, and for a compile. ⛔ **Never for answering a
person.** The map source is append-only, never corrected, possibly
stale and possibly self-contradictory — *rot is the feature*.

⭐⭐⭐ **The firewall is structural.** The four core modules —
`Traversal`, `KnowledgeGraph`, `TravelProfile`, `RoutePlan` — may
import nothing but each other, `GraphInvariants` and `MapClaim`.
⛔ Not `PlaceNode`, not the registry, not `DocumentApi`, not `StuffApi`,
nothing under `api/`. `lint:graph-walks`' second check enforces it with
**no ceiling**, and it earned its keep during the build: `TravelProfile`
imported `StoredEdge` for a *type* and was refused, rightly —
`PlaceNode` is a `Document`, so importing it puts the collection one
property access away from the module that must never consult the index.
It declares its own two-field `AdmissibleWay` instead, which
`StoredEdge` satisfies structurally.

A map planner that *could* reach the index would eventually consult it
— not maliciously, but because it was convenient once, in one branch.
The map **writer** (`Cartographer`) already had this property
deliberately; routing extends it to the reader, because a firewall
honoured by the careful is not a firewall.
`NavigationLogic.map-routing.test.ts` runs every case with the registry
absent and its prototype spied, asserting zero calls: the test is of
**reach**, not of behaviour.

⭐⭐ **The one join a map plan may make.** An edge claim records
`toLabel` — the far side's authored row path, read off the exit the
walker was standing at — and a singleton place's durable handle **is**
its row path. So they are equal whenever the far place is a template
node, and matching them is a string comparison *inside one document*.

⚠⚠ Where two claims disagree about one `(place, dir)`, **both edges are
admitted** and the plan names the disagreement. Claims append and
nothing is corrected; resolving a contradiction in the reader would be
the reader overriding the writer.

### A plan is a hypothesis

`RoutePlan` carries `nodes`, `legs` (each with its own `dir`), `cost`
on every axis, and **`assumptions`** — a first-class field, because
both sources can be wrong and the defect would be a plan that did not
say so. A map assumption cites the planner's own evidence (`channel` +
`lastSeen`); ⛔ nothing in it comes from the index.

⭐⭐ **Cost is quoted in the currency the traveller will pay.** The plan
measures every axis and the **renderer selects**: a walker is answered
in legs and what is in the way, a conveyance in minutes. That is
pedagogy, not presentation — `logistics.md` keeps ordinary movement
**instantaneous and free** on purpose, so quoting a pedestrian *"about
forty minutes"* for a walk the world will charge nothing for teaches a
figure that does not exist. ⚠ A pedestrian seeing a minutes figure is a
drive failure.

⭐ **Incomparable plans both come back.** One `cheapest-first` walk per
axis — minutes, legs, and `conditional` (⭐ a **risk** axis: the first
place in this game where a fast uncertain way can be weighed against a
slow sure one) — de-duplicated, and the engine does **not** pick when
they disagree. Choosing is the activity. ⚠ Stated plainly: that is a
*subset* of the Pareto front, one optimum per axis, honest for a realm
with three corridors.

⭐⭐ **The mode break.** When the mode-admitted search finds no way, the
same search admitting every medium says whether a way exists at all. If
it does, the first leg the traveller refuses becomes `breakAt`, and the
refusal reads *the way stops at the quay; north needs water* rather
than *there is no way to the island*. Only one of those tells you to
buy a boat. The second pass's plan is discarded — a plan built with the
omnivorous profile would send a wagon into a river.

### What the edge had to learn

Routing from the index was impossible until the stored edge carried
`media`, `wheelPassable` and `conditional` (§ The collection): the
projection could **cost** a wagon's route but not say whether a wagon
may *take* it, which is why a lane used to be compiled by walking live
exits one at a time.

⚠⚠ **A flooded ford is now in the lane, and that is the design.** The
old lane compile called `refreshCrossing()` by shape and skipped a
crossing the river had closed; the index is a projection of authored
rows and cannot know the water level. So the ford is in the lane, the
plan carries *this way is not always passable* as a stated assumption,
and the closure is discovered **at the leg**. A cached compile is a
worse place to learn about a river than the bank of it.

### `route` — the verb

`route to <place> [by <mode>]`, and `route between <a> and <b>` for what
a pair costs. Afforded by `Avatar.commandContributions.self` beside
`map`, for the same reason: both read the **player's** own document, and
`Perceiver` is composed on NPCs too. An NPC may keep a map — a
cartographer should — and gets no verb, because NPCs do not type.

It plans and **stops**: nothing started, nothing spent, nobody moved.
⚠⚠ Every place is named by what it **calls itself**, from the claims,
with the path leaf tidied into words as a last resort — never a
template path and **not a bare leaf** either. This verb consumes
handles and row paths end to end, which is exactly the shape that once
printed `/world/terminus/...` at a player.

⭐⭐⭐ **A map cannot answer `by wagon`, and says so.** A claim records
the *channel* you learned a way on and not what you were driving, so
your own map knows a way exists and cannot know whether a cart fits
through it. The two alternatives are worse: reading "no media recorded"
as "a footpath" refuses every conveyance with a mode-break sentence
about needing `ground` (nonsense to a reader), and inventing an
assumption would have the engine telling you something your map never
recorded. So it refuses in words and **names the verb that knows** —
`journey`, which plans with the vehicle's own declared mode, honestly,
because you are standing next to the vehicle. → A `walked` claim that
recorded what you were driving is designed and deferred to
`map-slate`.

### `costMatrix` — the matrix, and deliberately no optimiser

All-pairs cost between named places: the input a travelling-salesman
solver needs. ⭐⭐ Shipping it **without** the solver is the decision
rather than the omission — the engine computes what the roads cost and
the player decides which stop to make first, and taking that away would
take away the game. It answers `null` for a pair with no way rather
than omitting the row, because an absent row reads as *I did not ask*.
`route between <a> and <b>` is its typed reader, so the capability does
not ship with nobody able to see it.

### The census, and zero

`lint:graph-walks` counted **eleven** hand-written walks on an
untouched tree and holds the number at **zero**:
11 → 9 (the invariants, the grid) → 5 (the four perception walks) → 2
(mine air, the forage radius) → 0 (the lane compile, and `planRoute`
retired outright with no deprecated forward). Full detector, its two
checks, and the three false positives that sharpening it cost:
lint-family.md § `lint:graph-walks`.

⚠ Two callers keep their **own** neighbour reader, and both for a
reason: mine air reads `getExits()` — *all* of them, because a hidden
heading still has air in it — and the forage census admits a
destination that is not a `Container`, which the shared guards refuse
(a bee flying into a room-shaped nothing contributes no bloom but still
spends a hop, at the right distance). Routing either through the shared
reader would quietly change a number.

---

## Cross-references

- [location.md](./location.md) — the durable handle's contract; the Warren graph
- [boundary.md](./boundary.md) — exits, doors, the traversal gates
- [persistence.md](./persistence.md) — the `(scope, key)` spine; the registry's three reads
- [identity.md](./identity.md) — the two identity patterns; the mint census
- [parcel.md](./parcel.md) — title, and `published`
- lint-family.md — `lint:location-graph`'s counts and ceilings; `lint:graph-walks`
- [logistics.md](./logistics.md) — the lane, the Journey, the cost surface
- [perception.md](./perception.md) · [senses.md](./senses.md) — the four walks that ride `Traversal`

---

## History

The build landed on `reqs/location-graph` (MR !335) across ten waves —
A0–A4 (instance addressing) then B0–B4 (the graph, `published`, the
map) — between `288691af3` and the sweep. The requirements doc retired
at the sweep; the plan is **kept** for its deferred-seams section, and
both slates moved `builds/` → `tails/` once their substrate shipped
(the folder derives from `Size`).

⭐⭐ **Four things in this doc are the shape review argued it into, not
the shape it was built in.** Each is worth knowing before changing it
back:

1. **Map-keeping is `CartographerMixin`, not `Avatar`.** ~220 lines
   lived on `Avatar` behind a private predicate that re-narrowed the
   host set — this repo's own signal that a host is wrong.
2. ⛔ **The two `Perceiver` hooks were deleted.** `onPerceivedPlace` /
   `onReadTimetable` had ONE implementer between them, and an optional
   hook forces a structural cast at every call site — which is where
   three controllers' casts came from. The perceiver calls the recorder
   **directly** (`MixinApi.isCartographer`), the shape
   `BeliefStore.learnIdentityOf` already used. *A hook earns its keep
   by having more than one implementer*; `Mobile.onTraversed` does, so
   it stays one.
3. ⛔ **The claim is NAVIGATIONAL.** Channels went
   `perception`/`publication`/`told`/`bought` → `walked` · `seen` ·
   `searched` · `published`. The old axis covered looking AND walking
   with a comment apologizing for it, `told`/`bought` had no writer,
   and `modality` was a hardcoded `'vision'` nothing read beside a
   `band` nothing wrote. ⚠ The tell was in the renderer:
   `MapController` translated `channel === 'perception'` into the
   literal word *"walked"*. **A view layer compensating for a stored
   value means the vocabulary is wrong.**
4. ⭐⭐ **`searched` exists because non-obvious is not permanently
   absent.** A concealed exit is filtered out of `obviousExitsFor`
   until the viewer discovers it, so a glance that records no east exit
   is **not** evidence that there is none — and the staleness render
   had been asserting that it was. A deliberate search is the one
   observation whose absences carry information.

⚠ Two things recorded as owed rather than claimed: `search`'s
completion does not fire in a dev world (upstream of this build — the
completion's own message never appears, and `search.test.ts` cannot see
it because it force-advances the clock), and the only shipped concealed
exit is in `/world/lounge`, which resolves no Locality and so cannot be
mapped at all. So `searched` has a writer but no live end-to-end
consumer yet.
