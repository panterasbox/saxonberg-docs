# Logistics — goods move over real ground, by somebody, on paper

The realm had teleportation and no roads. Every `destination:` in every
shipped locality was checked during the seeding conversation: **zero
exits crossed a locality boundary.** Terminus, Hinkley Hills, Rejection
and newbie-wilds were islands stitched together only by TPA terminals,
and the freight slate's premise — *"you do not author roads, you author
which exits admit `wheeled`, and the road network is the induced
subgraph"* — had no graph to induce from.

This build delivers both halves: **real ground between the settlements,
and an industry that moves goods across it for money.**

Two packs own it, split on the `/system/` test — *a system is true
whether or not anyone is participating in it*:

| pack | root | what |
|---|---|---|
| **`transport`** | `/system/transport` | the ways. A road and a river exist with nobody employed by them. |
| **`trade-haulage`** | `/trade/haulage` | the hauling. Practised by somebody, and quittable. |

---

## The lane: an edge set, induced

**You do not draw a road.** A `Lane` names a `LocomotionMode`, and the
lane *is* the subgraph of exits that admit it — every edge whose
`allowsMode(mode)` holds, and for a wheeled lane whose `wheelPassable`
bit is set as well.

```yaml
# /stuff/idea/Lane/spine
key: spine
mode: wheeled
seeds: [/world/terminus/wharfside/bank]
operator: null            # the public highway
```

⭐⭐ Which is why **the pass works**. `the-pass` sets
`wheelPassable: false` on its exits (and `upper-climb` on the edge into
it), and the wheeled lane stops there of its own accord: a wagon reaches
the valley crossroads and cannot reach Rejection. **Bulk breaks at the
crossroads**, which is the depot's one shipped product doing real
economic work on the day the road opens. One bit on one exit, and an
economy.

⚠ **Every road exit must declare `media: [ground]` explicitly.** An exit
with no `media` is the legacy walk-only default and admits the ground
*pace* family alone — so a corridor authored the ordinary way has a
`wheeled` lane that compiles **completely empty**, with nothing anywhere
saying why. This is the single likeliest thing for a corridor author to
get wrong.

⚠ `seeds` is where the induced walk starts, not a map: rooms load
lazily, so *"walk every reachable room's exits"* has to begin somewhere
nameable. One authored path per lane buys the whole subgraph.

⭐ `edges[]` is the escape hatch for a lane with **no exits to induce
from** — rail, and the TPA. Authoring edges is what makes *"rail is a
data addition"* a fact rather than a hope: a realm that wants a railway
ships a row and no code.

⚠ **The operator is a ref, and may be nobody.** `null` is the public
highway; a path or `GroupRef` is a corpo-run or authority-run lane. Rail
and the TPA are incumbent networks in this design, and nothing in the
lane or `Route` shape may assume a player.

### Three lanes ship

| key | mode | note |
|---|---|---|
| `spine` | `wheeled` | the wagon's road, Terminus → the crossroads. Stops at the pass. |
| `estuary` | `sailed` | below the confluence only — the gorge is authored as *"steep enough that no boat has ever been up it"*, and the lane inherits that rather than restating it |
| `city` | `walk` | the whole of Terminus, induced with **no content change at all** |
| `ferrow-tram` | — | ⭐⭐ **the AUTHORED-edge case**: rails are not doors, so there is nothing to induce from and the edges are written down. The proof that *"rail is a data addition"* — a realm ships a row and no code. |

---

### ⚠⚠ There is no `tpa` lane, and deleting it is the lesson

One shipped as *"the limit case — teleportation is a lane with stop
density and duration driven to zero"*, and the claim is still true and
still worth making. The ROW was a mistake:

- **The terminals already are the graph.** A `TpaTerminal` carries
  `routes:` with **fares**; the lane restated the same three legs with no
  fares, naming ROOMS where the terminals name TERMINALS — two encodings
  of one fact, at different granularities, in different packs.
- ⭐ **And nothing read it.** Every brain that plans a route names
  `city`; `journey ... via tpa` cannot work because `journey` requires a
  vehicle and there is none in a terminal hall. So the two records could
  disagree forever and no consumer would ever notice.

⭐⭐ The demonstration it was carrying — *an authored-edge lane, for ways
that are not walkable exits* — moved to the **Ferrow tramway**, where it
is load-bearing: a tram sits on the rails and a player can travel them.

🔬 **Also logged, not done:** stop filing a bill for a settled gig at all
and DERIVE freight-from-gigs on read, the way this codebase derives
almost everything else. It would delete a whole write path. ⚠ The cost is
the coverage property: a bill is path-keyed under the filing business's
own branch, so *private books do not aggregate* — and a settled contract
lives in a global collection instead. That is a data-model change and it
wants its own conversation.

🔬 **Logged, not done:** *derive* a TPA lane from the terminals' own
`routes:` over the `TravelNode` shape, so the network has one record and
`journey`-over-the-Line becomes coherent if it is ever wanted. That is a
change to how lanes compile, and it needs the terminal→room mapping the
two encodings currently disagree about.

## The Route: nodes plus a stop set

A **value object**, never a Stuff. ⚠ That is a decision: a per-request
trip would otherwise mint a Stuff with no template path —
unaddressable, un-editable, and exactly the anti-pattern `lint:census`
exists to catch.

⭐ **Express versus local is ONE LANE with two stop sets.** An edge may
pass through nodes it does not stop at, which is what lets people at a
crossroads watch traffic go by without boarding it, and what means the
region never has to be carpeted in rooms.

⚠⚠ **The Journey cannot tell which factory made a Route.** A scheduled
service's route is authored (`ServiceRoute` rows in the commons); a
haulage gig's, and any on-demand trip's, is computed per request by
`LaneCatalogue.planRoute`. They are the same shape, and nothing may
branch on `provenance`. Today the path is unique so it does not bite; if
`Route` baked in *authored*, on-demand carriage would become
unrepresentable later and expensive to retrofit.

⚠ `ServiceRoute` (a data `Idea`) and `Route` (the value object) are two
things sharing a word, and the split is load-bearing: an author needs a
ROW to write, and a Journey needs a VALUE that mints nothing.

---

## The Journey: a sustained engagement whose beat is one leg

Not a `DurativeActivity`: a journey's duration is not trustworthy up
front, and a fixed timer would lie about when you arrive. So **the order
is a command and the advance is the scheduler.**

⭐⭐ **Every beat issues the same `traverse` a player's `go` does.** That
is the load-bearing rule and the reason there is no second movement
implementation: mode gates, `Exit.canTraverse`, the conveyance ripple and
the haulage tow all run on the shipped path, and a journey cannot
silently bypass a gate. The vehicle is genuinely present in every node
it passes, which is what makes a busy road visibly busy and an
interception possible with no interception system.

**Who moves, and who is engaged, are different objects:**

| shape | the mover | how the rest follows |
|---|---|---|
| a towed rig | the **driver**, hitched | the shipped haulage tow inside `Mobile.traverse` |
| a barge, a coach | the **vessel** | the shipped conveyance ripple carries occupants |

The engagement lives on the **driver** and holds the **`hands`** slot
only. `body`, `attention` and `voice` stay free, so **a driver cannot
fight back** and **a passenger holds no engagement at all** — which is
what makes an escort mechanically necessary rather than merely sensible.
⚠ `combat` is deliberately **not** in `interruptibleBy`: being shot at
does not stop your wagon; stopping is the driver's own `cancel`.

### The metronome and the score

The framework fixes an emission's interval at start, so the beat is a
**one-game-minute tick** and each leg spends a budget:

```
legGameMinutes = edgeMinutes × modeFactor(mode) × loadFactor(rig)
```

⭐ `Exit.edgeMinutes` is **game minutes for one baseline unloaded
walk-mode traverse of that edge**, defaulting to
`transport.defaultEdgeMinutes`. ⚠⚠ **Nothing in the kernel reads it** —
`go north` stays instantaneous, and it must: a duration on ordinary
movement would be a real-time toll on every step in the game.

⚠ It is authored per EDGE rather than derived from `coords` distance,
because **length here is an event budget, not a metric**: a lonely
stretch is few rooms and long edges, and coordinates are grid membership
rather than a distance.

**Arrival is a completion**, through `SchedulerApi.complete` — added by
this build, because `cancel` used to be a sustained engagement's only
exit and *arriving* and *being stopped* were therefore the same event in
the envelope.

⚠ **Blocked means blocked.** A journey that cannot take its next leg
aborts `route-blocked` and leaves the vehicle in the node it reached.
There is no auto-replan, deliberately: auto-routing hides the geography
this build exists to make real.

---

### ⭐⭐⭐ Fault tolerance: what breaks an in-flight journey, and how it ends

A Journey holds a **route snapshot, a leg index, a vehicle ref and a
driver ref**, and every one can go stale. The per-leg transaction
boundary is where they are re-checked — *"a journey that checked once at
the start would drive through a door somebody shut an hour ago"*.

| state change | how it ends |
|---|---|
| the driver is **displaced** (teleport, `goto`, carried off) | `displaced` — and they are **not** dragged back |
| the driver **leaves a self-propelled vessel** | `displaced` — a barge nobody is steering does not sail on |
| the driver **dies or collapses** | `driver-incapable` |
| the vehicle is destroyed | `vehicle-disabled` |
| the hitch breaks (breakaway, unhitch, a teleport severing it) | `vehicle-disabled` — never *"the driver walks on with the cargo standing in the road"* |
| an exit is blocked, removed, or its gate now refuses | `route-blocked` |
| the lane is recompiled under it | `route-blocked`, via a leg that no longer joins |
| the server restarts or the pack reloads | the journey is simply **gone** — engagements are deliberately non-persistent, so the driver is parked in a real room with every slot free |
| the driver is attacked | ⭐ nothing — `combat` is deliberately absent from `interruptibleBy`. Being shot at does not stop your wagon; stopping is the driver's own `cancel` |
| a passenger disembarks | nothing — a passenger holds no engagement |

⚠⚠ **Displacement was the dangerous one, and the reason is worth
keeping.** `Mobile.traverse` takes its origin from the **exit**, not from
the mover. Every other caller resolves its exit from the room the mover
is standing in, so origin correctness is *structural* for them — and a
journey is the only caller that holds an exit **across time**. A beat
firing while the driver stood somewhere else moved them from wherever
they were to the far end of a road they had left: a teleport, dressed as
a step.

⭐ A NON-LIVING driver is never `driver-incapable`, on purpose: an
automaton pulling a cart does not die at the reins.

## The vehicles

⭐⭐ **`VehicularMixin` is what makes them one kind.** A wagon, a barge
and a coach are three unrelated compositions — `Haulable` and
deliberately not `Mobile`; a `Drivable` vessel that steers itself; a
sealed carriage — and identical to a traveller: something you take
somewhere. The pack's own mixin (`/system/transport/lib/Vehicular`)
carries the `journey` affordance, the residency veto, and the marker
`MixinApi.isActive` narrows on.

⚠ Before it, that identity was spelled three times and guessed a fourth:
the affordance copied onto each class, the eviction veto on two of them
(**a parked wagon was cullable and a parked barge was not**, for no
reason anybody chose), and `JourneyController` re-deriving *"what counts
as a vehicle"* as `isHaulable ‖ (isDrivable ∧ isMobile)`. ⭐ A guard that
re-narrows the host set is a mixin trying to exist.

⚠ It is the PACK's mixin, never the kernel's — the kernel must not know
`journey` exists. `collectBucketDefs` reads `commandContributions` *"off
the class and every mixin in its chain"*, so the affordance is collected;
and narrowing uses the marker string with `isActive`, not `hasMixin`,
because `MixinName` is a closed kernel union and **a pack must never
need a kernel list edit**.

| | composition | moves by |
|---|---|---|
| `HaulageRig` | `Bulkable(Haulable(Vessel))` | **towed** — and deliberately NOT `Mobile`. A wagon is pulled, and the shipped tow already carries it and its cargo as a unit. |
| `Barge` | `Bulkable(Drivable(Slotted(Mobile(Vessel))))` | self-propelled |
| `Coach` | `Drivable(Sealable(Mobile(ExitableVessel)))` | ⭐ the consumer `ExitableVessel` had been waiting for since the boundary build |

⭐ **Perception out of a vehicle needs no code at all.** Passengers are
*contents*, and `MixinApi.isOpenContainer` is the single rule `canReach`,
the MQL `peers` walk and `VisionModality` all ask. An open wagon shows
you the road; a shut `Sealable` coach does not. A per-row `data`
decision.

⭐ **Variety is data, not subclassing.** The wagon, the dray and the
sledge are one class: `draftFactor 0.03` against `0.35` is the whole
difference, and the encumbrance substrate turns that one number into
everything a player notices.

### ⚠ Two composition traps, both found at class-stamp time

- **A `Coach` must not compose `SlottedMixin`.** `ExitableVessel` gets
  one through `Adornable`; a second throws `FinalViolationError:
  overrides final method SlottedMixin.occupyAll` before a line runs. A
  `Barge` over a plain `Vessel` needs its own.
- **A bare coach row must not compose `SeatedDrivableMixin`.** The
  seated override finds a driver-role seat OBJECT in the vessel's
  contents and *throws* when there is none, so a coach shipping no box
  seat would refuse to be driven the first time anybody tried.

### Handling: competence is ACCESS, never a better outcome

A rig may carry `requiredDiscipline` + `requiredBand`, and refuses
through `Haulable.canHitch` — a new optional hook, consulted by shape,
because *refusing to be coupled is a question about the cart*.

⚠ **The ACTOR's competence decides, not the hauler's**: a horse has no
transcript, and the person putting it in the shafts is the one who does
or does not know how.

⚠⚠ **The transport system never interprets the key.** The wagon's ROW
says `teamstering`; the class hands the string to the actor and reads a
band back, exactly as `Drivable` is handed a `vehicularMode` path. A
content word in the code would have been the mistake — and would have
made `transport` depend on the pack that depends on it.

⭐ **Band 0 must be able to earn.** The barrow, the handcart and the
sledge ask nothing of anybody, and the handcart is stocked at Terminus
inside a starting stipend. A labor market whose smallest job needs
capital is not an entry rung.

---

## The paper

Three document kinds, all rows in `documents` — **no new collection** —
path-keyed under the filing business's own branch, all `onVanish: 'keep'`
because each is *a record of something that happened*.

### The bill of lading

*What, how much, from where, to where, whose, at what declared value* —
plus the route's legs.

> It is not a reporting feature. It is the paper a carrier needs to
> prove what they took and a shipper needs to prove what they sent —
> and it happens to be the datum every freight statistic reads from.

⭐⭐ **It is what makes a fungible shipment nameable.** The gig
substrate's `Condition` refuses `Stackable` outright ("a merging stack
has no stable identity"), and supply needs are overwhelmingly fungible.
A gig for "twenty bottles" is unpostable — so the consignment is a
**discrete, chattel-stamped crate**, and the bill says what is in it. No
new condition template, no vocabulary edit, no engine seam.

⚠⚠ **It is filed by the CARRIAGE, not by the verb.** `ship` at a
counter, the `hauls` brain, and **a player who claimed a gig and
delivered it** all file the same paper — the third through the
issuer's `onContractSettled` `@hook`, which names no trade. ⭐ It was a
global `contract.settled` bus event with one emitter and one subscriber,
which cost the kernel an `Events` entry, an interface shaped around this
pack's fields, and an `emittableBy()` policy left OPEN so anything could
forge the announcement; the substrate now tells **the issuer and nobody
else**, and `CarrierBusiness` overrides. D16 makes the gig the
dominant path, so paper filed only by the first two would leave the whole
reporting spine blind to most freight in the realm.

### The warehouse receipt

A bailee's acknowledgement, and a **document of title**: you transfer the
receipt instead of moving anything.

⚠⚠ **It is a RECORD, and there is deliberately no object.** It shipped
with a bearer/registered split — a `BearerReceipt` `Thing` you could be
robbed of, versus a row naming a person — and the bearer half was **cut
before merge, because it was proof of nothing**: no `withdraw`, no claim
check, nothing anywhere read the slip back. Holding one entitled the
holder to nothing and stealing one accomplished nothing. A prop with a
strong docstring attached, which is the same failure as a help example
that cannot be typed.

⭐ **The missing seam, named:** `withdraw <receipt>` at the shed —
checking who holds what, handing the goods over, voiding the row. Build
that and the bearer instrument comes back with a reason to exist and
real persistence semantics to get right; the asymmetry (*one of them can
be lost*) is a good design and it needs the act underneath it. Built
apart, it is furniture.

### The rate card

> **Rates must be visible and settable.** Visible, because rate
> discrimination is the antitrust arc's evidence and must be a table
> rather than an accusation. Settable, because a carrier that cannot
> choose its prices cannot be the villain of that arc.

⭐⭐ Visible means **a board on the wall**, not a verb on somebody's
books: `house` is your own books and would have been exactly the wrong
shape. `RateBoard` composes the shipped `MarkedMixin`, so reading is a
thing everybody can already do (and a board in the dark is a blank,
which is correct). Its text is **derived on read** from the current card.

⚠ A new card never overwrites the old one. **A superseded card is
exactly the row an antitrust argument needs.**

⚠ **Two pricing mechanisms coexist and price different acts**: the
**card** is the carrier naming a price in advance and serving all comers
(you tender at a counter); the **gig board** is the shipper naming a
price per job. Both are real and historical; neither is a degenerate
case of the other.

### Reporting is a query over the paper

`WaybillRegistry.freightOf` reads **by prefix under the business's own
branch**, so a depot's records cover exactly what it handled and no
others — coverage IS market share, structurally. `trafficOf` counts the
legs.

⭐⭐ **Edge traffic is derived from bills of lading and no traffic
counter is stored anywhere.** Nobody authors that the valley road is
busy; it is busy because the ore goes down it, and if the mine closes it
stops being busy on its own. ⭐ And the gap between what a road was
BUILT for (authored: mode gates, grade, the edge budget) and what it
CARRIES (arithmetic) is the story engine — a fine road nobody uses, a rut
carrying more than it should.

⭐ The honest consequence of shipping without customs: **private books
do not aggregate.** Nobody sees the realm's trade, only their own, and
the first institution that can see across is not the state — it is the
depot.

---

## The labor market

### ⭐⭐ Which verb moves a thing? Four answers, and they are different markets

| you want | verb | the market it is |
|---|---|---|
| to take it yourself | `journey to <stop>` | no market — your time, your rig, your risk |
| a carrier to take it, for a posted tariff | `ship <goods> to <place>` | a **freight company**: you hand over custody, get a bill of lading, and pay the rate on the board |
| *anyone* to take it, for a price you set | `job post deliver … for <n>` | a **spot market**: escrow up front, first to satisfy it is paid |
| *anyone* to bring you N of something | `job post supply <n> <kind> … for <n>` | the same spot market, denominated in QUANTITY |
| somebody in a seat, ongoing | `appoint <who> to <position> at <org>` | **employment** — a wage, a roster, a chart |

⚠ `job` and `appoint` are the pair most often confused, and the names
point the wrong way: *"job"* sounds like employment and *"appoint"*
sounds ceremonial. It is the reverse — `appoint` fills a seat, `job`
posts piecework.

### The condition vocabulary is CLOSED, and that is the wall

A gig may only be posted if its condition is a member of
`CONDITION_TEMPLATES` — today `delivery` and `supply`. The rule is not
tidiness: **escrow holds real money from the moment the gig goes up**, so
completion must be checkable by the engine at turn-in with nobody
adjudicating. *"Bring me ten iron ore"* is checkable. *"Guard my shop"*
and *"be nice to Mara"* are not, and are refused rather than
half-supported.

⭐⭐ **`supply` is why there is no extraction template**, and the
reasoning generalises. *"Go mine ten iron ore"* looks like a condition
about mining; it is not. Verifying that YOU mined it needs provenance on
every lump — expensive — and it is bad economics besides, because it
would forbid filling the contract by **buying**, which is a legitimate
way to fill one and the thing that makes a spot market liquid. **How you
sourced it is your business.** Mining, salvage, purchase and hoarding
collapse into one predicate the engine can count.

⭐⭐⭐ **And a contract says what the BUSINESS wants, not what the author
imagined it arriving in.** A supply condition may bind a *category and a
unit* — `6 L of gin` — instead of a template path, and the tally then
MEASURES rather than counts: six litres is six litres whether it comes
in one demijohn or eight bottles.

⚠⚠ That is a player-agency fix, and the bug it replaces is the sharp
version of a mistake this build made twice. A business is denominated in
**category and unit**; everything it handles is denominated in
**objects**; and translating between them is where things went wrong —
first when `exemplarFor` read a bottle's *vessel kind* as its par
category and no bulk line ever matched, then when a gig bound a template
path so *eight of that exact row* was the only way to satisfy it. A
player who brought one demijohn holding six litres had done the job in
every sense the bar cares about, and the engine counted zero. ⭐ The
matcher now lives in ONE place (`CategoryMeasure`), and the par sheet,
the keeper's exemplar scan, the carter's crate finder and the contract
tally all read the same answer — there were four copies, and two were
wrong in different ways.

⭐ The condition is a **phrase**, not a flag and not a subcommand:
`post`/`claim`/`complete`/`abandon` are what you are DOING, the phrase is
what the work IS — two axes, two slots. A third template adds a phrase
form and changes nothing else. It is also what retired `--kind`: one
grammar was doing two jobs and needed a flag to say which, and now
`deliver <that crate>` and `supply 1 <kind>` are two sentences.

⭐⭐ **The build's second purpose, and the pattern every NPC-run sector
will reuse.** Hauling has no skill floor, universal demand, and is
perfectly parcelable — so it is the first labor market that can be
transferred from NPCs to players.

> **Post the work, let players take it, let the NPC cover the
> residual.**

- **`restocks`** posts what the rail is short of and **receives** what a
  hauler leaves on the bench. The hauler buys at the supplier and is
  reimbursed on delivery, so the distributor is still paid and the
  consignors still see their resale.
- **`hauls`** (the carter) covers any posting nobody took inside its
  window, on the **same `Journey` object** a player drives.
- **`consigns`** ⚠ **walks**, and does not post. See below — the reason
  is the money, and it is the build's one named economic seam.

### ⚠⚠ Why the two brains ended up different shapes

The plan expected both to post, because *"every producer floor is an
exitless island"* made walking impossible. **This build removed that
reason** — the floors have doors onto the goods yards now — so the
question became which answer is honest, and it is not the same answer
for both:

| | can it walk? | what it does | why |
|---|---|---|---|
| `consigns` | **yes**, since this build | walks to the counter and consigns | consignment is **sale-or-return**: the producer is paid *on resale, out of its own listing*. A crate hauled there by somebody else is a crate **nobody has listed**, and no shipped mechanism lets a carrier — or the distributor's clerk — list goods on the producer's behalf. Posting this leg would have quietly stopped paying six producers. |
| `restocks` | **no** | posts, and receives | its host is the Lounge bar, and *"Saxonberg and the Lounge joining the map"* is a stated non-goal. Its leg is closed by the hauler **buying and being reimbursed**. |

⚠ **The surfaced seam: there is no wholesale purchase.** A distributor
cannot buy a producer's goods outright, and a carrier cannot list goods
it is only carrying. Until one of those exists, a producer→distributor
leg cannot be a posted gig without silently unpaying the producer. It is
one mechanism — `consign`-as-another-house, or a wholesale `buy` at the
works — and it belongs to whichever build wants NPC producers to stop
walking. **It is named rather than faked**, and the walking version
regresses nothing.

⚠ Consequently the labor market's **first rung is the venue-supply leg**,
which needs enough float to buy one line. That is small but not zero, and
it is honestly weaker than the plan's intent (a pure-carriage job needing
no capital at all). The porter gig that would restore it is D16's, and it
arrives with the wholesale purchase.

Three things that buys at once:

1. the economy stays **DAU-independent** — the NPC always covers;
2. every NPC haul is **visibly a job a player could have taken**;
3. ⭐⭐ **the NPC is the reserve supply, so it sets the wage.** A player
   cannot charge more than the carter costs and need not accept less.
   The reservation wage, doing real work. Its rate is authored data with
   its reasoning beside it (`haulage.npcRatePerKgMinor`), never a
   constant somebody picked.

⚠ Postings are `--bounty` with **no expiry**: no claim step, escrow held
from post, anyone may turn it in — so a player who gets there first is
paid and the carter finds nothing. And a posting that lapsed would revert
the escrow and leave the venue unstocked. **The window is the carter's
patience, not the posting's lifetime.**

⚠ **The order lands on a receiving BENCH, not on the shelf.** A gig is
refused when its condition already holds, and a short line usually still
has something on the shelf — so an order aimed at the shelf would be
refused precisely when the venue most wanted it. The bench is empty by
construction.

⭐⭐⭐ **A venue orders what it has NONE of — which is the whole point.**
An order names an item, and `job post` resolves that name the way its
`destination` always has: reachable-first, falling back to reading the
string as a **kind's path**. That fallback is not a convenience. A bar
ships with an empty rail on purpose (nothing is spawned into a shelf),
so on a fresh realm every supplied line is at literal zero — and a
keeper who can only point at what she has could order nothing, forever.
**Dave's Bar could not open.** The par line's `exemplar` names the kind,
which is the proprietor's decision anyway: *which* gin this bar stocks.

⚠ A gig for a kind that is nothing would hold its escrow until somebody
abandoned it, so `ContractApi.post` refuses one — a live instance at the
path is proof enough, the template row is the fallback.

⚠⚠ **A bounty with no expiry needs two guards, and both are load-bearing:**

- **`job post --kind`.** A posting binds to the exemplar's *instance*
  when the exemplar is chattel-marked (deliver THIS crate) — the right
  reading when you point at your own goods, and the wrong one when a
  keeper points at a bottle on her own rail to say what she wants more
  of. Every bottle on a venue's rail is marked, because the venue bought
  it. `--kind` forces the kind reading, and it is an ordinary
  player-facing distinction rather than an NPC affordance.
- **A kind already pending on that bench is not re-ordered.** The line
  stays short until somebody carries it and the bounty never lapses, so
  without this the keeper escrows another reward every beat until the
  house is broke. ⭐ *One beat looks perfect* — which is why a fixture
  that runs a single beat, and a live drive whose window is minutes, can
  both miss it. The test asserts the **second** beat.

⭐ **The backhaul is visible.** `jobs --origin here` lists work that
STARTS where you stand, so a hauler at the far end can see what wants
moving back. *You cannot solve your own backhaul* — you need somebody
else's cargo going the other way — which is a coordination problem with
visible waste, and the cleanest teachable case of **why intermediaries
exist**.

---

## The forcing function

`consigns` and `restocks` **stop teleporting**, and the resolution is
not that they walk:

> **`restocks` became a poster and a receiver.** It never leaves the bar.
> **`consigns` WALKS** — over the road this build authored, through the
> back door this build gave its floor. Neither teleports.

Two shipped facts and one non-goal ruled the literal reading out: every
producer floor was an **exitless island**, and `restocks`' host is the
Saxonberg Lounge bar, which *"Saxonberg and the Lounge joining the map"*
keeps off the road.

⭐ For the keeper that is the better reading anyway: not *"she walks four
rooms"* but *"she does not travel, because carriage is somebody's job."*
For the producer's hand the plain reading became available the moment
the door existed, and the money says to take it — see *"why the two
brains ended up different shapes"* above.

**What it cost:** one new room (the goods yards, behind Wharfside) and
one door from each mainland producer floor onto it. The hands never use
those doors; the haulers do. ⭐ The Lounge stays off the map and its leg
rides the **TPA lane** — D2's own limit case doing exactly the work D2
says it does, and diegetically right: the Compact's seat is served by the
Authority's network.

⚠ **The par sheet was retuned as part of the decision**, not left to
discovery: every supplied level doubled when the road took over. *A bar
that cannot restock because the road is slower than the drinking is a
regression, not a lesson.*

---

## The corridors

Two shipped, plus the one that was already there.

| corridor | rooms | edges | reads as |
|---|---|---|---|
| the Delight road | 5 | long | ⭐ **the lonely stretch** — the ford, the milestone, the drove crossing, the empty flats, the crossroads |
| the estuary | 3 | short | ⭐ **the towpath** — same total duration, completely different road |
| the Kestrel road | 5 | long | the climb, the LAST WATER, **the pass**, the tips, the yard gate |

⭐⭐ The two-corridor contrast is D18 made checkable: **length is an event
budget, not a distance.** *If a corridor has five rooms, five things have
to be true there.*

### The ford — a road that changes with the season

`FordExit` reads `flowAt(reach)` — **the same number `measure` reads** —
against a threshold and sets the shipped `blocked` bit. No new field on
the water, no new mechanism, no weather of its own.

⚠ It reads the water pack **by shape**, never by import (the
`AnalyzeWaterController` / `TravelNode` idiom): an install with no water
pack has a ford that is simply always passable. It memoises on the
catalogue's own **six-game-hour weather segment**, so a ford and
`measure` cannot disagree about the same water at the same moment.

### ⚠⚠ Never put a direction word in a detail's keywords

`go west` resolves its argument through MQL, and a detail keyworded
`west` **wins the match over the exit** — the walk then answers *"you
can't reach the valley crossroads"* and the realm quietly stops being
contiguous. Found by driving; it cost two rooms and would have cost the
build's first acceptance criterion.

### Land use: no seventh entry

A corridor is **subdivided as its own parcel**, so the road across
somebody's flats is the road's ground and not theirs. `civic` (the
public highway) and `wild` (unserviced ground — passage and gathering
only) already carry it, and ⭐ the `civic`/`wild` split **is** the
toll-versus-obstruction distinction: a gate on ground you hold is
legitimate, a barricade on the commons is not.

### Addresses, and the jurisdictional gap

The Delight road sits under `terminus/delight-valley/…`, which no
`Locality` claims — so `AddressApi.coverageChainOf` returns an empty
chain and *"banditry lives in jurisdictional gaps"* is derivable today,
authored rather than asserted. The Kestrel road is Rejection's; the
estuary is the city's waterfront. **One corridor covered, one not**, and
the contrast is the point.

---

## Archetypes

⚠⚠ **A corridor is asked over its LANE, not its zone.** `surveyScope:
corridor` reports the archetype across a whole way at once, and the
question *"which rooms are the way?"* was first answered with the room's
zone — because the kernel could reach a zone and could not import the
transport pack. That is the unit that was reachable, not the unit that
was right: a one-room outdoor courtyard reported a corridor that is a
spot, and a city-sized zone would report *the corridor has water*
because a fountain exists a quarter-mile away. The shipped roads read
correctly only because they happen to be zoned one-road-per-zone.

⭐ It resolves `LaneCatalogue.lanesAt(here)` **by shape** now — the
duck-typed singleton lookup `consigns` already uses — so the kernel
still imports no pack, a crossroads on two ways is asked about both, and
**outdoor ground that is on no lane is asked nothing at all**, which is
the behaviour a courtyard should have had from the start.

⭐ `lanesAt` is a **keyed read** off a `byNode` index built during the
compile (`place → the lanes running through it`). It was a filter over
every lane in the realm, recomputed per depot per question; `allLanes`
is private now, because a public whole-table read invites the next
caller to narrow it too. See
[antipatterns.md § An Api May Not Hand Back Its Table](../antipatterns.md).

Two substrate fields, and the second is not optional:

- **`materializesOnto`** — ⭐ an archetype builds a **wagon**, not a
  room. `materialize()` always returned `Stuff & Container`; the only
  thing binding archetypes to locations was one constant.
- **`surveyScope`** — `space` (today's behaviour) · `corridor` (the
  rooms of the zone you stand in, at once) · `off-room` (never reported
  from a room). ⚠ Without it, five new industry-less archetypes would
  print on **every `survey` in the game**, in every bedroom, for every
  player.

⭐ `corridor` scope is gated on **`BiomeApi.isSkyExposed`** rather than a
zone marker: *outdoors is what makes somewhere a WAY rather than a room*,
it is a shipped kernel predicate, and it costs no field anywhere (D20:
*zone, nothing new*). ⚠ An outdoor square does answer the corridor
questions — and that is right, not noise: D18's own table calls the high
street a corridor.

**The roster:** `corridor` (many spaces) · `depot` (premises) ·
`haulage-rig` and `passenger-conveyance` (conveyances) · `livery` — ⚠
**the file ships, the content does not**; ranching brings the stable.

⭐ *Reported, never enforced.* A corridor with no shelter is a legal,
visible state and the report names the gap. **No ninth need was added** —
shelter is `seating`, water is a `bulkSource`, a crossing is a
`presence`, light is `lightLux`.

---

## The teleport defect, fixed

⭐⭐ **Coupling does not block a teleport; a teleport severs it.** The
first fix REFUSED — a hitched hauler or a seated rider could not
teleport at all — and that was correct about the data and wrong about
the world. It made the spell feel broken rather than the wagon feel
heavy, and *"you cannot teleport while holding a rope"* is not a rule
anybody would write on purpose.

What happens instead is the honest thing: **you go, and what you were
attached to does not.** The hitch slips, the wagon stands where it
stood, the saddle is left; both sides are told, because a player who
teleports away from their wagon and is not told has been robbed by a
mechanic, while one who is told has made a choice.

⭐ Freight still does not teleport, so the cost surface is untouched —
and nothing is lost, because a parked vehicle vetoes residency eviction
(`VehicularMixin`), so the wagon is there when you walk back for it.

⚠ The severing is what the refusal was really protecting: a one-sided
move would leave the coupling dangling at one end and lying at the
other. `unhitch` and `Slotted.vacate` are the chokepoints, so both ends
come apart together.

> **Teleport ripples what is *on* you and severs what you are *attached
> to* — and says so.**

Worn gear and a pack are contents and come along; slot occupants ripple
(a mount teleporting brings its rider); a **hitched** hauler and a
**mounted** rider are *coupled* by a live-ref pair the move cannot
carry, so the coupling is slipped and both sides are told. The wizard
`goto` behaves identically — relocating a coupled object leaves what it
was coupled to standing, which is what "relocate this object" should
mean.

---

## ⭐⭐⭐ Routing: what the pathfinding actually is

**One BFS, per lane, over a compiled adjacency map.** `LaneCatalogue`
induces each lane's edges once, caches the graph (dropped on HMR), and
`planRoute(from, to, lane)` breadth-firsts across it, bounded at 2,000
nodes so a mis-authored graph cannot hang a boot. `consigns`' walk and
the carter's `journey` both call it.

Three properties of that, stated because each is a decision:

- ⭐ **It is hop-optimal, not time-optimal.** `edgeMinutes` weights the
  DURATION of the trip, never the search. A road with fewer, longer legs
  wins over a shorter one with more legs. Roads are roads; nobody
  Dijkstras a valley.
- ⚠ **There is no cross-lane routing.** A lane is a mode, so a route
  that would change modes — wheels to a back at the pass — is *not
  planned for you*, and that is the design: **breaking bulk is the
  lesson**, and auto-routing around it would hide the geography this
  build exists to make real. Blocked means blocked.
- ⭐ **There is no general pathfinding Api and should not be one yet.**
  A general graph search invites auto-routing at every call site, which
  is the regression above. When a second edge set needs search, promote
  the walk then — not before.

Cost is not a concern at this size: the compile is once, a query is
O(V+E) over a bounded, cached map, and the callers are a player's verb
and an NPC beat measured in minutes.

## ⚠⚠ Players and NPCs do not travel the same way, and that hid every bug

An NPC brain drives the **literal verb** — `hauls` issues
`journey to <path> via <lane>` through ordinary dispatch, subject to
every gate a person is. That is the doctrine and it is right. But it
passes a **template path it computed**, and a player must *say where*.

> **An NPC brain driving a literal verb proves the verb's MECHANISM and
> never its GRAMMAR.**

Every place-naming defect in this build shipped behind that gap and was
invisible to a green suite: `ship`'s destination could not name a remote
place at all, `journey`'s had the same flaw copied from the same
comment, and the drive typed neither. The NPC loops looked healthy
because they were — the naming layer is player-only, and nothing
exercised it. ⭐ A drive that does not TYPE a verb has not tested it.

The other asymmetries, which are experience rather than defect: a
driver's hands are occupied for the whole trip (you cannot fight while
driving, which is what creates the escort economy), passengers board
with `go <vehicle>` and leave with `out`, and a blocked road stops you
where you stand. An NPC experiences none of this: `travel()` is one
forced command that either arrives or does not.

## ⭐⭐ The cost surface is OPT-IN — decided, not overlooked

**Ordinary movement is instantaneous and free.** `Exit.edgeMinutes` is
spent only by the Journey engagement — the kernel never reads it, and
`AppSettings` says so in as many words. So:

> A player who types `go north` eleven times — or writes an `msh` script
> that does — crosses the same ground in **zero game time**, with a
> hitched rig and its cargo following, and pays none of the cost surface
> this build is about.

That is not a bug in any file; it is the mechanic being **opt-in**, and
scripting makes opting out free. It is named here rather than papered
over. ⭐ **Decided at review (2026-09-04):**

1. ⛔ **Charge the edge in the kernel** — ordinary traversal of an edge
   with `edgeMinutes` costs that time. **REJECTED, on a principle worth
   reusing: *moving around must not feel like netlag.*** A player
   walking a corridor is navigating, not waiting; a delay on every `go`
   reads as the server being slow rather than as the world being big,
   and no amount of correctness buys that back. ⚠ Note this rejects the
   *universal* form — it does not reject cost per se, which is what
   option 2 charges to the load rather than to the walk.
2. 🔬 **Make the rig the gate** — a laden or hitched vehicle may only
   cross a corridor edge inside a Journey. Narrow: it leaves walking
   untouched and charges only what freight does, which is the half that
   the cost surface is actually about. **Parked pending playtest**, not
   rejected. It cannot be judged from a design doc because the question
   is how it FEELS across the modalities — on foot, mounted, driving a
   rig, aboard a barge — and whether the gate reads as the cart being
   heavy or as the game saying no. That playtest is the deciding
   evidence and it has not been run.
3. ✅ **Accept it** — the shipped behaviour. `journey` is a convenience
   (one command, passengers, a readable trip, a vehicle really in every
   room) and freight speed is not a hard constraint.

⚠ The tension is accepted knowingly rather than resolved: *"a wagon is
cheaper per kg over a graded road"* is a claim about cost the world does
not currently enforce against a player willing to type `go` repeatedly.
That is survivable while the corridor economy is young, and it is the
first thing to revisit if freight speed ever needs to mean something.

## What is deliberately not here

- **Customs, tariffs, checkpoints** — blocked, not deferred: there is no
  statute engine to enact them with. → the border build.
- **Rail, timetables, the ore train** — the lane substrate is general
  enough that rail is a data addition; no rail ships.
- **Tollgate, barricade, banditry, congestion, road wear** → the
  contested road build.
- **Warehousing as a business** — rent, shrinkage, the stocktake. Its
  spine is *record versus reality*, and nothing in scope makes the two
  diverge, so a stocktake would always balance. ⚠ And **capacity is a
  property of a bearer's BODY**; a warehouse has no body, so *full* is
  unrepresentable. What ships is the receipt and the duty.
- **A passenger service.** The taxonomy is recorded and the `ride`
  experience ships; the market does not exist, because walking is free
  across a five-minute basin. ⭐ The one rung with real demand today is
  the **porter**, and it is D16's entry gig.
- **The wainwright.** You can own a wagon; you cannot make one.
- **Navigation as a discipline.** ⭐ A discipline needs somewhere to get
  lost, and a realm with two corridors has no wayfinding in it.

## Cross-references

[conveyance](./conveyance.md) · [locomotion](./locomotion.md) ·
[spatial](./spatial.md) · [boundary](./boundary.md) ·
[location](./location.md) · [address](./address.md) ·
[watershed](./watershed.md) · [encumbrance](./encumbrance.md) ·
[activity](./activity.md) · [contract](./contract.md) ·
[chattel](./chattel.md) · [credential](./credential.md) ·
[document-store](./document-store.md) · [retail](./retail.md) ·
[employment](./employment.md) · [attendant](./attendant.md) ·
[behavior](./behavior.md) · [advancement](./advancement.md) ·
[content-packs](./content-packs.md) · [parcel](./parcel.md) ·
[time](./time.md)
