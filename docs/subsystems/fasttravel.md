# Fast travel (the Teleport Authority)

The teleport-based transit network — content-named the **Teleport
Authority (TPA)**. A directed graph of public **terminals** you teleport
between, gated by a **scan-to-register** credential and now **powered by
mana it has to buy**. Distinct from [locomotion](./locomotion.md)
(physical room-to-room movement) and from [boundary](./boundary.md)
exits: fast travel is discontinuous hops across an authored network, not
stepping through a door.

## ⭐ What the TPA reform changed, and why

Teleportation used to be free, distance-priced by an author's intuition,
and structurally the kernel's. Three things moved:

1. **Teleportation is a SPELL** whose price is physics. `m·g·Δh`, and
   **distance appears nowhere in it** — the hard part is *specification*,
   not distance, so the spell's authored `cost` prices the survey and the
   physics prices the lift. See [magic.md](./magic.md) and
   [arcane-science.md](../arcane-science.md).
2. **The network is a UTILITY selling a capability its customers do not
   have.** It runs on mana, from a cell or a city line or a traveller's
   own reserve, and it resells that mana at its cost basis — so a
   frontier post on bought cells quotes a dearer ride than a city gate on
   the line, and nobody tuned that number.
3. **The NETWORK is not the kernel's — but the VERB is.** The works live
   in the `tpa` capability pack (`/system/tpa`); the socket lives in
   `arcana` (`/system/arcana/lib/ManaPowered`); the kernel keeps the
   physics (`MagicApi.relocationCost`), the cast pipeline, and the
   `teleport` verb itself.

   ⭐⭐ **You must not need the Teleport Authority to teleport.** Moving
   yourself around inside an extent you hold is *authorial authority* —
   the same authority that lets you edit the place — and casting a
   working that relocates you is *magic*. Neither is a transit network's
   business, and neither may stop working because a content pack is
   absent. The reform got this wrong on its first pass: the whole verb
   moved into the pack, so on a platform-only boot a privileged person
   had no way to move at all, and `AuthorMixin` — a KERNEL mixin — was
   left naming a pack's view.

   So the verb is kernel and the network is a pack's, meeting over the
   **`TravelNode` shape** (`lib/travel/TravelNode.ts`) — plain data in,
   plain data out, never an import. Water's `SupplyReporting` is the
   same seam — a waterworks is read by the shape it answers, and the
   asker never names its class.

## Where the code is

| | |
|---|---|
| `packages/server/src/mud/lib/travel/TravelNode.ts` | **the seam** — the shape a travel network answers, declared in the kernel |
| `packages/server/src/mud/platform/idea/cmd/movement/TeleportController.ts` | **the verb** — free movement + the anchored spell, delegating the ride and the board |
| `packages/content/tpa/src/lib/FastTravel.ts` | the node mixin — routes, board, timetable, `boardLabel`, and `ride()`: everything the NETWORK decides |
| `packages/content/tpa/src/thing/TpaTerminal.ts` | the concrete terminal: a node that runs on mana |
| `packages/content/tpa/src/thing/TravelCard.ts` | the carryable instrument |
| `packages/content/tpa/src/idea/cmd/movement/` | `RegisterController` |
| `packages/content/tpa/content/system/tpa/` | `register` + `procure card`, the card, the Authority |
| `packages/content/arcana/src/lib/ManaPowered.ts` | the socket every mana-powered device draws from |
| `packages/server/src/mud/api/magic.ts` | `relocationCost` — the kernel's `m·g·Δh` |

⚠ A pack cannot add to the kernel's `Mixins` registry, so narrowing is
`MixinApi.isActive(x, FAST_TRAVEL_MIXIN)` against the exported marker,
not `MixinApi.isFastTravel` (which is deleted). The terminal rows keep
their `/world/**` paths — **a terminal belongs to the locality it stands
in; the mechanism belongs to the system.**

## The two halves

**`FastTravelMixin`** (`lib/fasttravel/FastTravel.ts`) — a **node**
(a TPA terminal): a public-room fixture (a `Thing`) that routes to other
nodes. It owns a **directionality** (`arrival` / `departure` / `both`),
a directed set of **routes**, a **selected destination** (state), an
advance policy, an inert `status` seam, and `getArrivalRoom()` (where
travellers land — the node's own container).

The concrete `TpaTerminal` (`/system/tpa/thing/TpaTerminal`) renders this
directionality as a **diegetic status light** — the *color of the
terminal's name* in a room listing (purple both-ways / blue arrival /
red departure / grey out-of-service), via a `getPresentationMml`
override that wraps the name in a `<color>` tag (see
[message-rendering.md](./message-rendering.md)). The colour carries the
meaning; the long description spells the same condition out in words
(the non-colour-alone channel) on a deliberate `look`. The reform added
**amber** — running, but short for *this* ride — and made the grey
band mean something (below).

⭐⭐ **`LoungeTerminal` is gone.** It was a subclass carrying exactly two
overrides: a board label, and a `getArrivalRoom()` that was already
redundant (the row's `seatIn` makes `FixtureMixin.seatSelf` move the
terminal INTO the live Warren host, so the generic `getContainer()`
already answered). `FastTravelMixin` gained an authored `boardLabel`
instead — a bespoke string is content, not a class — and the equivalence
test was written *before* the delete, not after.

**The credential** is now a `travel` **record** held in a
`CredentialWalletMixin` holder (the unified credential substrate — see
[credential.md](./credential.md)): a registered-node set plus the
register/authorize surface. The holder is base-agnostic, so it composes
around two bases — the Thing/Idea symmetry of the three-base capability
model (see [augmentation.md](./augmentation.md)):

- a carryable **`TravelCard`** `Thing` (`/system/tpa/thing/TravelCard` =
  `CredentialWalletMixin(Thing)`) — a bearer **instrument**, never a
  clearance store (see **Identity-bound clearance** below);
- the born-with **`CredentialWalletUpdate`**
  (`CredentialWalletMixin(AetherHostedMixin(Idea))`) — the one wallet app
  injected into the avatar's aether attunement by
  `Avatar.installDefaultLoadout`, holding the `travel` record alongside
  `payment`. This is the **identity holder**. The `AetherImplant` confers
  attunement only and composes no credential mixin.

## Identity-bound clearance (the card is an instrument, not a passport)

Travel **authorization** — the registered-node set — is a structural
property of the *traveller's identity*: it is read from and written to the
actor's own **aether-hosted wallet** only, never a carried, transferable
card. The split is deliberate:

- **Instrument gate** ("do you have the means to use the TPA at all?") —
  the MQL `reachable` pool filtered on `isCredentialWallet` + a `travel`
  record. A carried `TravelCard` **or** the born-with implant satisfies
  it, so onboarding and the un-implanted are never stranded.
- **Clearance** ("are you cleared for *this* destination?") — a direct
  single-object read over the actor's own `getHostedUpdates()` (one
  level), so clearance is bound to *who the actor is*. `register`
  writes here; the `teleport` fork's `isRegistered` check and
  `renderDepartures`' "not yet registered" annotation read here.

The identity read is the reachable walk's self-hosted leg in isolation —
identity-only, single-level, a plain read on one object (project
doctrine: single-object reads are direct). The security property is
**structural, not scan-order**: handing a loaded card to another player
confers no clearance they didn't already hold on their own identity (the
card's own registered set is never consulted for authorization). The
`TravelCard` keeps a floored `travel` record purely so it satisfies the
*instrument* gate.

*Edge (documented, not engineered around):* an un-attuned actor with no
hosted wallet has no identity clearance store; `register` fails gracefully
("nowhere to record"), and the ride is refused as not-registered.
Effectively every player avatar is attuned and carries the born-with
wallet.

## The network model

- **Directed routes, per-terminal authoring.** A `TravelRoute` is one
  directed edge: a destination **node by singleton path** (`ref`) plus an
  optional per-route world-clock **timetable** (`departures`). Routes are
  an **instruction field** (`routes`) applied once at hydrate. Whoever
  places a terminal wires its destinations (like authoring an exit); the
  MUD-wide network is the sum of those choices — **no central planner.**
  Not all-to-all; you travel only where a terminal routes.
- **The node is its own source of truth.** Every destination read goes
  off the **live destination node** (`StuffApi.singleton(route.ref)`),
  never off template data. A node's network identity is simply its own
  singleton path; the registered set is a set of those paths.
- **Cascade load.** `armNetwork()` resolves each route's destination to a
  live singleton, which cascades — the whole reachable network loads from
  a single boot-manifest root.
- **Public infrastructure, last mile on foot.** Terminals live in
  lobbies / stops / hubs, never private/instanced space. You fast-TP to
  the dorm lobby, then walk to your room. (Authoring discipline, not an
  engine rule.)

## Targeting and the departures board

- **`teleport <keyword>`** — the raw token is matched **locally** against
  *this* node's routes (`resolveRouteByKeyword`, a live keyword read off
  each destination node), **not** MQL world resolution. Ambiguous keyword
  → asks for specificity; no match → "no route here goes to …".
- **bare `teleport`** at a node renders the **departures board**
  (`renderDepartures`) — a live, viewer-aware list of routes with the
  now-boarding marker, per-route times, a `⊙N` **fare** tag for paid
  routes, a "not yet registered" note (read off the viewer's **identity**
  wallet) for destinations they haven't unlocked, and a line saying
  **which way this viewer's mana will be paid for**.

  ⭐⭐ **The board is a public timetable, not a boarding pass** — it
  renders for EVERYONE, before any clearance read (D9/AC15). This closes
  a defect the doc carried from 2026-08-01: the board was unreachable
  from a bare `teleport` for its whole life, for two independent
  reasons. An extent-holder self-powered past the TPA fork before it ran,
  and an unregistered traveller was refused by the registration gate
  before it rendered — so the only people who could see the routes were
  the ones who already had them. It is fixed by DECIDING ON "was a
  keyword typed", not on `getSelectedDestination()`: that getter falls
  back to the first route and is therefore never null on a node with
  routes, which is why the old `if (!ref)` arm was dead code.
- **Destinations are named by covering Locality (D13).** The board shows
  each destination's covering-`Locality` name via
  `FastTravel.getDestinationLabel()` — in order: the authored
  **`boardLabel`** when set, then resolve the node's arrival room's
  address (`AddressApi.resolveLocalityFor`) → `Locality.getName()`, then
  the terminal's own presentation. The lounge authors `boardLabel: The
  Lounge` because its Warren host carries no stable address for the walk
  to land on. Display-only: **keyword targeting stays on the
  terminal's authored `keywords`** (`teleport terminus`/`crossroads`/`lounge`),
  not the multi-word Locality name. The three board Localities (Terminus /
  The Lounge / The Last Counted Mile) are seeded under `seeds/lib/address/`
  so `AddressRegistry.postRegister` warms their prefixes at boot.
- **Selection** is node state: `selectedDestinationRef` (defaults to the
  first route). A keyword sets it; the timetable can flip it.

## The verbs

**`teleport` / `tp`** — **the KERNEL's verb**
(`platform/idea/cmd/movement/TeleportController.ts`,
`platform/cmd/movement/teleport.yaml`), afforded by `MobileMixin`
beside `go` and `goto`. One verb, and a fork order that decides which of
four things you meant:

```
both endpoints in one extent you HOLD → free, no network, no magic
at a travel node, a stop named        → the node's ride
at a travel node, nothing named       → the node's board, for EVERYONE
otherwise, a destination named        → the anchored spell
```

⭐⭐ Forks **1 and 4 are kernel** and run on kernel capabilities alone;
forks **2 and 3 are the network's**, reached over the `TravelNode`
shape. ⚠ **A node does not GRANT the verb** — it adds the ride and the
board to a verb everybody already has. That is what lets a privileged
person teleport in a world with no travel network installed.

⭐ **The order is load-bearing.** It used to ask `canSelfTeleport` FIRST,
which is what made the board unreachable for both audiences at once.
Deciding the board before any clearance read closes it.

⚠ **Object relocation is not here.** `teleport --target` became the
kernel's **`goto --subject`**: authorial tooling must not evaporate when
a content pack is absent, and one verb meaning two unrelated things
depending on who typed it was the wrong shape anyway. `goto` was already
in the author category and already did the focus walk.

Only `requiresAnimate` is verb-level. The two TPA-shaped validators
(`requiresTravelCredential`, `mustBeAtFastTravelNode`) were **deleted**
with the pack cut: they lived in the kernel's validator directory, which
is exactly where a capability pack's preconditions must not live, and the
controller re-checked both redundantly on every call.

### 1 · Free movement inside an extent you hold (D11)

Both endpoints inside ONE extent you hold ⇒ the hop is free: no mana, no
fare, no registration, no terminal.

⭐ **The frame is AUTHORIAL AUTHORITY, not privilege.** The same
authority that lets you edit the place lets you be in it — *moving around
inside what you author is not a journey.* Cross the boundary and it is
the TPA like everyone else, and the wizard axis (code trust) buys no
movement at all.

⚠ A **use-grant is not authorship**, and it is excluded *structurally*:
`AccessApi.heldExtents` admits on `ParcelRecord.getOwner()` and never
reads `grants[]`, so a lease-holder gets none of this and nothing had to
be written to exclude them. See [access.md](./access.md).

### 2 · The TPA ride — `FastTravelMixin.ride()`

⭐ **Everything the network decides lives on the node**, with the pack:
the keyword match, the instrument gate, identity-bound clearance,
direction, the gate's own condition, the mana leg and the fare. The
kernel's verb knows none of it — it finds something answering the shape
and asks.

The keyword is matched **locally** against this node's routes, never
through MQL. Then, in order: the instrument gate (any reachable travel
holder) → departure-capable → **the gate's own condition** (below) →
clearance (the identity holder's `isRegistered`) → **the mana leg**
(below) → fare settlement → `Mobile.teleport` to the destination's
`getArrivalRoom()`.

⚠ The **one** refusal the kernel falls through on is
`route-not-found` — the network not going somewhere is not a refusal of
the traveller. Every other refusal is final, and is rendered in the
node's own words.

⭐⭐ **The ride issues no `prepareCast` and no `resolveCast`.** It quotes
`MagicApi.relocationCost`, draws that many τ off the gate, settles the
money, and moves the traveller. That is D10's *"the TPA is a utility
selling a capability its customers do not have"* expressed structurally:
a ride through the cast pipeline would inherit the band gate, and **the
network's entire customer base would be locked out of the network.** The
two paths share the **cost function**, not the pipeline.

### 3 · The anchored spell (D10)

At no node — or at a node whose network does not go where you asked — a
caster casts instead. ⭐ Falling through rather than refusing is the
honest order: *the terminal is a convenience, not a permission.*

**You may only teleport somewhere you can honestly be said to hold in
mind**, and there are exactly three ways to hold a place. First hit wins:

1. **an extent you hold** — you authored it. One `<extent>/**` path-glob
   seed per held extent, each through `MqlApi.resolveMany`. This is what
   `CommandLogic`'s `tries` loop does with a view's static `scope:` list;
   the difference is that the list is computed PER ACTOR, which a static
   YAML `scope:` cannot express — hence controller-side.
2. **a registered node** — you physically went there and surveyed it. A
   small enumerated set of template paths off the credential, matched by
   keyword against each live node. No MQL, no scan.
3. **current scope** — `here`, then `peers`, then `reachable`.

⚠⚠ **There is no fourth anchor, and never a `world:` seed.** Resolving
is not permission, and a world scan would quietly turn "somewhere you
hold in mind" into "anywhere that exists" (see
[mql.md](./mql.md) and the world-scan-perf slate). A test spies on the
SCOPE ARGUMENT across every branch including the miss, so a future anchor
cannot slip one past.

⭐ **Ambiguity is a FAILED SPECIFICATION, never a disambiguation prompt.**
"Which of these did you mean" is the engine admitting you did not hold
the place clearly and then arriving anyway — and specification is the
whole difficulty. The view carries `onExcess: error` so no other path can
silently pick a winner, and the specification is checked BEFORE the
faculty gate: a vague name is a failure about the place, not about you.

The anchored hop **refuses a short pool** rather than overchannelling
into strain. Overchannelling is a fair price for a firebolt you chose to
force; paying it to travel would make strain the ordinary cost of getting
anywhere.

ⓘ Two front doors, deliberately complementary: **`cast teleport` is
see-it-and-go** (arcana's verb, the ordinary `reachable` scope, a hop
across a room you are looking at); **`teleport` is the anchored door** —
the long hop to somewhere you are not. One spell row, one cost function,
two grammars.

**`register`** (`RegisterController`,
`system/tpa/cmd/movement/register.yaml`) — records the terminal here onto
your credential. **Contributed by the terminal itself**, so it surfaces
only when you're at a node; the afforded node is `context.commandSource`.
Arrival-capable nodes only, and **free at every node** (there is no
`settle` call reachable from it).

⭐ `teleport` used to be afforded only by `AuthorMixin`, so the
"unprivileged TPA ride" the controller always documented was reachable
by authors and nobody else. It is on `MobileMixin` now — a movement
verb, afforded to anything that moves.

## Unlock = scan-to-register

You reach a node **by other means first** (walk, story), `register` your
credential at it to add it to your allowed set, *then* you can fast-TP to
it. Explicit, diegetic — not auto-unlock. The TPA fork checks
`cred.isRegistered(ref)` before travelling and refuses with "reach it
another way and `register` first".

**Born-with floor:** every credential is born registered for the nodes
the **authored** `fasttravel.bornWithNodes` names — the **Terminus
arrival node**, **the lounge**, and **the paid destination** (the
newbie-wilds crossroads) — the documented exception to
"reach-before-travel". The hub has no foot path to the rest of the world, so
the interchange and the social hub are universally reachable by design:
onboarding's lounge→campus hop lands at Terminus (walk across the avenue to
campus), and the lounge floor node makes Gate A's free return to Dave's Bar
work even for a fresh player who never explicitly registered it. The paid
destination is on the floor because the **fare**, not registration, gates it.
The floor is preserved across hydration (the `TravelCredential` record's
`fromData` unions saved entries on top of it, never clearing).

⭐ **It is authored DATA, not a code constant** (D12/AC22). It was three
`/world/**` paths hard-coded in `lib/credential/Credential.ts` — content
paths in the kernel, which is exactly what the pack cut exists to
remove. *Which* of a realm's stops every traveller is born knowing is a
REALM decision, so the values live in `world-seed`; the code fallback is
**empty**, so a kernel with no teleport pack has an empty floor and is
correct rather than born registered for stops that do not exist.

## ⭐⭐ The gate runs on mana

A `TpaTerminal` composes **exactly what a dorm room's wall lamp
composes** — `ManaPoweredMixin` over `Slotted + Charged`. Neither is a
special case of the other, and the lamp shipped FIRST on purpose: a
capability only one class composes is a method wearing a costume (AC6).
See [magic-items.md](./magic-items.md).

Read the composition bottom-up and every layer is forced:

| layer | why it is there |
|---|---|
| **`ConduitMixin`** | ⭐ the most load-bearing find in the reform. `MagicLogic.bestConduitFor` scans the actor's ENVIRONMENT for a conduit and refuses a transfer without one — *"bare hands are a poor road for that much energy."* A terminal that composes it **is its own coupling**, so a traveller pours their pool in with no rod, no bench, and zero kernel change. Already true in the fiction: the terminal is a brass pillar, and brass conducts. |
| **`ChargedMixin`** | an impulse device *draws per use off a stored charge*. The reservoir IS the impulse shape — it is what makes `dry` mean something, what `chargeFrom` needs a shell for, and what makes three supplies three ways of filling ONE thing rather than three parallel draw paths. |
| **`SlottedMixin`** | the `battery` bay. `accepts` may only name a KERNEL `Mixins` value, so the bay takes any charged shell and `ManaCell.fitsSlot` narrows candidate-side. |
| **`ContainerMixin`** | a part that goes into a machine has to physically BE somewhere. The slot is occupancy; containment is location — the `plant`-into-a-pot order. |
| **`ManaPoweredMixin`** | the draw surface and the condition. |
| **`PersistableMixin`** | the reservoir level and the bay occupant round-trip through `holder_snapshots` and the shipped `SlottedSlice`. **No collection was added** (AC26). |

### The three supplies, and the one branch

A **cell** in the bay, a **`ManaMain`** the row names, or a **person in
contact**. All three resolve through one `resolveSupply()`, and **the
device holds no branch on which answered** (AC8). The contact arm routes
through the shipped `chargeFrom`, which already refuses a non-caster —
so AC12's negative half falls out with nothing written: a non-caster
holds no reserve at all, because `installArcaneReserve` returns early
for one.

### The condition, and why there are two reads

`status` used to be a persisted string that **nothing ever wrote**: an
author could grey a light and no mechanism in the game could. It now
DERIVES from `supplyState()`, in the shipped six-word supply vocabulary
(`dry` · `cut` · `frozen` · `fouled` · `off` · `overdrawn` — see
[watershed.md](./watershed.md), whose `SupplyState` this is the second
speaker of). ⚠ The vocabulary is **imported**; `SupplyReporting` is
deliberately **not implemented**, because `AnalyzeWaterController` reads
that shape structurally and doing so would make `analyze water
<terminal>` work by accident.

Two reads, and folding them is how the light ends up lying:

- **`supplyState()`** — the **stock** question. Knows no amount. This is
  what `getStatus()` and the light read.
- **`stateForDraw(tau)`** — the **transaction** question. Can it cover
  THIS ride?

### The arming floor, and the three bands (D8b)

`armingFloorTau` is a **separate fact** from `drawMode`: a wall lamp
authors `0` and is purely impulse; a gate authors a real floor and is
both. Nothing about `drawMode`'s meaning changes.

| band | light | what happens |
|---|---|---|
| below the floor | **grey** | `dry`. The gate refuses **even a BYO ride** — below its floor it is not a gate, and there is nothing for your mana to arrive INTO. |
| armed, short for THIS ride | **amber** | `overdrawn`. Refuses an unfunded ride, **accepts one you power** — and accepts a *cheaper* ride unfunded, which is what proves `overdrawn` is a **relationship**, not a property. |
| armed and covered | blue/red/purple | it runs. |

⭐ A gate with **no traffic at all** drains to `dry` over game-time
(`standbyWatts`, reconcile-on-read — no clock, the `GrowingMixin` /
soil shape, reusing `Charge.standbyDraw` verbatim). *Absence is a cost.*
That is what gives a cell swap a schedule, and what gives the Teleport
Authority its first real job.

### Whose mana pays (D8c)

The shipped three-tier chain, not a hand-rolled `resolveSetting(…) ??
'terminal'`:

```
flag (--channel | --meter)  →  the actor's `tpa.power` setting  →  terminal
```

⚠ A sticky preference that is **invisible at the moment it applies** is
the footgun, so the departures board — already viewer-aware — carries a
line saying which way YOUR ride will go. Visible, it is a convenience.

The BYO path calls `chargeFrom` **directly**, with no spell-knowledge
gate: **fuel is not casting** (D8). `recharge <terminal>` remains the
pre-loading route and keeps its extra `transfer` gate; both land in the
same reservoir, so the outcome is the same either way.

⚠ `ChargedMixin` affords `zap` as well as `recharge`. `TpaTerminal`
overrides the static to drop `zap` — a terminal is not a wand — and a
test asserts the collector reads the **most-derived** static rather than
unioning the chain. If that ever changes, the fix is `zap.yaml` growing
`requires: [ArcaneMixin]`, which is the honest gate regardless.

## The transit-fare economy

A route carries an author-set **`fee`** (`TravelRoute.fee`, minor units;
`0` = free — the lounge↔terminus hops stay free). A paid ride **settles the
fare before travelling** (`TeleportController.settleFare`); insufficient
funds **refuses the ride** (no partial charge, traveller not moved). The
fare **splits the operating budgets** over banking's remittance-split seam
(the first wired consumer), and **conserves** — a fare *moves* money, only
the CB mints:

- **The city's operating budget** (the bulk) — the `Business` operating the
  **departure terminal** (the *fixture*, not the room), resolved un-spoofably
  via `EmploymentApi.ensureOperatorAt(node)` — keyed on the terminal so two
  venues sharing a room each resolve their own operator, and standing the
  Business up lazily from its own `operatingLocations` if it isn't live yet
  (see [employment.md](./employment.md); no manifest entry, no standup hook).
  Its account collects `fee − networkFee`; a paid route with no operator is an
  authoring error → refuse.
- **The TPA's operating budget** — a **network fee** to the **Teleport
  Authority Business** (`fasttravel.tpaBusinessPath`, a minimal Business
  row at **`/system/tpa/idea/teleport-authority`**: proprietor-absent
  public-infrastructure operator, empty `operatingLocations` so it never
  collides with the per-terminal fare operators, resolved by path). The
  fee lands on its operating account at its authored `banksAt` (the TPA
  is **not the state** — only the treasury banks at the CB, so it banks
  commercially; every account names a real custodian, see
  [banking.md](./banking.md)).

  ⭐⭐ **It governs ITSELF**, and that is what let the row move out of
  Terminus and into the mechanism pack (D1/AC21). The network crosses
  every locality it serves; a network that one of its stops appoints to
  is *that stop's* network, and the others are guests on it. So the
  appointing authority is `{kind: committee, parcel: /system/tpa}` — the
  committee over the pack's OWN extent, which **names no realm**.

  Seats fill the standard capability-pack way with nothing to build:
  that committee → the `tpa` group the manifest's title claim gives it →
  `owner: {office: prime-minister}` (the same declaration `water` and
  `arcana` make) → the founder by Art. XI's pool-of-one, with the PM
  operator override on top. `Authority.ts` says the quiet part: only
  `office` and `committee` carry that default, so *an authority the
  founder cannot satisfy is one nobody can satisfy on a cold box* —
  which is why `committee` was already right here and a group would have
  been wrong. See [civics.md](./civics.md), [governance.md](./governance.md).

  And there is nothing to fill **yet**: `positions: []`, a
  self-regulating institution with no staff. ⭐ The **first** position is
  the one the arming floor creates — a frontier gate drains with zero
  traffic, so somebody swaps cells on a schedule, and that somebody is a
  TPA employee appointed by the TPA's own committee. It gives
  [vocations.md](../vocations.md)'s *water/sewer worker*-shaped gap an
  employer that is neither a locality nor a corpo: the first seat
  arriving with a reason rather than as scaffolding. Out of scope here;
  recorded so nobody invents a staffing model this does not need.

  ⓘ The kernel's ONE remaining piece of TPA knowledge — `BankingLogic`'s
  legacy raw `tpa` accumulator re-own — was **deleted** rather than
  moved: it was self-described migration, there are no users and no
  data, and the kernel has no business knowing the Teleport Authority
  exists. The fee is a **flat base + a percentage** —
  `min(fee, base + floor(fee × rate))`, tunable via
  `fasttravel.networkFeeBase` / `fasttravel.networkFeeRate` — the
  payment-processor shape, so the TPA **collects a non-zero fee on every
  paid ride** (a pure percentage floors to zero on micro-fares). An
  unseeded world (no TPA Business resolvable) waives the levy with a
  warning rather than blocking the ride.

**The mana charge (D8a) — and ⭐ the rate is DERIVED, not authored.**

`total = fee + surcharge + manaCharge`, where

```
manaCharge = ceil(costTau × node.manaRatePerTau())
```

and `manaRatePerTau()` reads off `getSupplyMode()`:

| supply | dial | seeded |
|---|---|---|
| the city line | `tpa.manaRate.mains` | `0.002` |
| bought cells | `tpa.manaRate.cell` | `0.01` |
| your own / none | — | `0` |

⭐⭐ **Neither number was chosen.** A cell is 6 credits for 600 τ at the
general store, so cell mana costs the operator **exactly 0.01 per τ** —
and that is what a cell-fed gate charges, because the operator resells
its own supply at its cost basis. The line is a fifth of that; the city
buys in bulk.

The consequence is legible without a gauge and is what AC13 asserts: two
otherwise-identical gates quote **different prices for the same ride**,
and the traveller can see WHY by looking at what is feeding each one. On
a **level** hop both rates round to a single credit and the mana is a
rounding error on the fare; on a real **climb** the `mgh` term dominates
and they visibly diverge.

⚠ A traveller who **channels their own** pays a mana charge of **zero**
(AC11) — you cannot be charged for mana the operator did not buy. That
is the same ride at two prices, and the difference is a physical fact
about who supplied the energy.

The mana charge is a fourth split leg payable to the **departure**
operator, the one that bought the mana.

**Arrival surcharge (destination-imposed).** On top of the route's `fee`, a
**destination node** can impose its own **`surcharge`** (`FastTravel.surcharge`,
minor units, optional like the fee) — a charge just for *using that terminal as
a destination*. The traveller pays `total = fee + surcharge`; the board shows
the total (`⊙total`, broken out `(fee+surcharge)` when both). The surcharge is
collected by the Business operating the **destination terminal**
(`ensureOperatorAt(destNode)`, fixture-keyed like the fee), the mirror of the
fee's departure attribution — so
each end's operator collects its own charge, un-spoofably, over the same
`settle` split (departure op keeps `fee − networkFee`, TPA takes `networkFee` on
the **fee only**, destination op takes the whole `surcharge`; conserved). A
`surcharge > 0` with no destination operator is an authoring error → refuse. A
fee-0 route into a surcharged destination is charged the surcharge alone (no
network fee — the ride itself is free). Demo: the newbie-wilds crossroads
("The Last Counted Mile") levies a small **arrival toll** collected by its
frontier settlement budget (stood up by the bespoke `CrossroadsTerminal`), so
Gate A → crossroads costs `15 + 2` — Terminus keeps the fare, the frontier the
toll — while the free return leg stays free.

Both budgets **recirculate** (the city budget pays the clerk's wages — the
conserved loop; the TPA budget funds network maintenance, *deferred*). The
sink lever is a **CB drain** from a budget back to the CB (a monetary-policy
decision, *deferred*), **not** passive accumulation. Settlement is
**tender-agnostic** (D12): the fare tries the payment credential, then
**cash** — a cash fare **crosses the cash bridge** (the coin is consumed,
the equal value credited on-ledger to the city budget and split to the TPA;
supply-neutral) so the split holds either way. P&L categories `fare` /
`networkFee` (`lib/banking/LedgerEntry.ts`).

## Terminus — the transit hub (network content)

**Terminus** (`/world/terminus/`, its own branch = the **Terminus
municipality** owner sphere, distinct from the EU campus) is the network's
central interchange, **on University Avenue across the street from the
university gate** (a cross-branch `across` exit to the EU plaza; the
standalone University Avenue terminal is **retired**, Terminus absorbs the
campus-arrival role). Its first zone is the terminal building
(`/world/terminus/terminal`): a station **hall**, an **arrival gate**
(directionality `arrival` — the floor node the lounge routes to), **three
departure gates** (**A operational** with the free lounge return + the paid
crossroads line; **B and C out of service**), and a **ticket office**. It is
the first content to exercise the `directionality` feature (every prior
terminal was `both`) and the **out-of-service** band — the ride fork
reads `getStatus()` at authorization and refuses a non-operational
departure.

⭐ Since the reform, the city gates are **line-fed** (`mainsRef` → a
`ManaMain` in the gate hall) and the frontier posts — Hinkley Hills and
the crossroads — run on **bought cells** with a real arming floor and a
standing draw. That is not decoration: it is why the same hop costs
different money from different gates, and it is the content that gives
the Authority a maintenance round to staff.

The four terminals are **not** boot-manifest entries — they load by
**`props:` cascade** from the single lounge root (`saxonberg-lounge`'s
`boot:` list holds one hub anchor, `/world/lounge/thing/terminal`, not one
entry per terminal). The three departure terminals
are never route targets, so a plain route-target cascade would never reach
them; instead each departure gate room declares `props:` its own
terminal fixture, so standing up the hub materializes every gate + its
terminal. The office's **terminal clerk** (`TicketClerk`,
"Tootie") is the city-budget `Business`'s paid employee — she **procures a
free replacement `TravelCard`** (`procure card`, the `tpa/` command
category; a card is an instrument, no fare) and receives **wages** from the
city budget at her shift boundary (the roster tick; a bounded shift so the
wage settles). `settleShiftWageImpl` gained a **worker-account guard** that
auto-provisions an NPC worker's account so the wage lands. The **paid
destination** is the newbie-wilds **crossroads** ("The Last Counted Mile",
`/world/newbie-wilds/crossroads`) — a minimal v0 (one designed landing room
+ one `both` terminal, free return), the integration anchor the deferred
newbie-wilds build extends.

New **committed non-guest** players receive a small **onboarding coin**
(`issueCash` at the char-gen commit; hard coin, no account, anti-farm ~20,
guests excluded) — spendable at the bar and the fare (both accept cash).

> ⚠ **Reseed hazard (R2) — live.** `fasttravel.tpaBusinessPath` is a
> `settings`-kind row, and that kind is **merge-missing**: an existing
> world keeps pointing at the retired `/world/terminus/terminal/idea/tpa`
> until an operator edits the key by hand
> (`config set fasttravel.tpaBusinessPath /system/tpa/idea/teleport-authority`).
> A fresh DB is correct automatically. Same shape as R1 below.

> **Reseed note (R1) — historical.** `SeederManager` *was* insert-only (the
> pack installer's reconcile has since replaced it), so on an *existing*
> world the repointed lounge route, the deleted UA terminal, the plaza edit,
> and the new `fasttravel.*` / `banking.onboardingStipend` AppSettings do not
> apply live: delete-and-restart the three affected `domain` rows and reboot
> (the AppSettings seeder merges the new keys on next boot). Fresh DBs are
> correct automatically.

## Timetable & advance policy

`advanceMode` ∈ `manual` / `scheduled` / `cycle`, bound to game time via
`WorldClockApi` (`armTimetable` / `disarmTimetable`, host-scoped clock
handles):

- **scheduled** — each route's `departures` (cron patterns, `HH:MM`) fire
  `selectDeparture(route.ref)` at those game times.
- **cycle** — `every(cycleInterval)` rotates the selection through routes.
- **manual** — selection only changes on a keyword.

## Persistence

**The gate's own state survives a restart.** `TpaTerminal` composes
`PersistableMixin` (a singleton, keyed on `templatePath`), and
`PersistableLogic` already captures a `SlottedSlice` plus per-mixin field
slices — so the reservoir level **and the cell sitting in the bay** both
round-trip through `holder_snapshots`, an existing collection. No
collection was added (AC26). A `ManaMain` needs nothing: it refills.

### Credential caveat (v1)

Credential registration is **session-durable, not cross-restart**:
`Avatar.save()` persists only the avatar's own fields (no inventory
persist-back; the hosted `CredentialWalletUpdate` is re-cloned each session
by `installDefaultLoadout`, so the `travel` record's registered set resets
to the born-with floor each login), so it does not survive a server restart
yet. Cross-restart durability rides future persistence work (aug-state
colocation; inventory persist-back) — see [persistence](./persistence.md).

## Seams & deferred surface

Captured in `docs/slates/tails/fast-travel-slate.md`:

- ~~**Living infrastructure** — the `status` seam is inert.~~ ✅ **Half
  closed.** `status` DERIVES from the gate's supply now, so a gate goes
  dark for a reason the world produced: it ran out, its line was cut, its
  cell was never swapped. What is still deferred is *wear* — a gate that
  degrades from use rather than from absence — and the Authority's
  maintenance ROUND as a staffed job. The Authority owns *standard +
  health*, never topology.
- **Cross-restart credential durability** (above).
- Richer targeting, fares/access tiers, and disruption events.

Two things found by driving the Hinkley Hills commute route
(2026-08-01). **The first is now CLOSED; the second still stands:**

- ~~**Nobody actually sees a departures board.**~~ ✅ **Fixed by the TPA
  reform.** The board now renders whenever no keyword was typed, before
  any clearance read, for everyone — and `teleport` is afforded by
  `MobileMixin` rather than by `AuthorMixin`, so the unprivileged ride
  the controller always documented is finally reachable by the
  unprivileged.
  The design question the old note declined to answer is answered:
  **a timetable is public; a ticket is not.**

  ⚠ Worth keeping for the lesson: the defect had TWO independent causes
  and fixing either alone would have left it. One was the fork order; the
  other was that `getSelectedDestination()` falls back to the first route
  and is therefore never null, so the `if (!ref)` arm that rendered the
  board was dead code from the day it was written.
- **⚠ A stale terminal row silently deletes a route, and the refusal
  lies.** A terminal row that the pack reconcile leaves alone (a locally
  edited row is a *conflict*, never an overwrite — `pack status` names it)
  keeps the live network on the old list. `teleport <kw>` then answers **"no route here goes to '<kw>'"** —
  which is *also* what an unauthored route says. The two are
  indistinguishable in a transcript, and the good outcome (the
  registration gate) is the one that proves the route is really wired.
  When a route is added to an existing terminal, the row has to be
  re-seeded — content-pack reconcile does this, a bare dev restart does
  not.
