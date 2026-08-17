# Fast travel (the Teleport Authority)

The teleport-based transit network — content-named the **Eternal City
Teleport Authority (TPA)**. A directed graph of public **terminals** you
teleport between, gated by a **scan-to-register** credential. This is
**distinct from [locomotion](./locomotion.md)** (physical room-to-room
movement) and from [boundary](./boundary.md) exits: fast travel is
discontinuous hops across an authored network, not stepping through a
door. The lounge→campus hop and char-gen/onboarding home-routing ride
it.

Code lives in `lib/fasttravel/` (the two mixins), `obj/command/author/
TeleportController.ts` + `obj/command/movement/RegisterController.ts`
(the verbs), and `cmd/{author/teleport,movement/register}.yaml` (the
views).

## The two halves

**`FastTravelMixin`** (`lib/fasttravel/FastTravel.ts`) — a **node**
(a TPA terminal): a public-room fixture (a `Thing`) that routes to other
nodes. It owns a **directionality** (`arrival` / `departure` / `both`),
a directed set of **routes**, a **selected destination** (state), an
advance policy, an inert `status` seam, and `getArrivalRoom()` (where
travellers land — the node's own container).

The concrete `TpaTerminal` (`domain/common/tpa/`) renders this
directionality as a **diegetic status light** — the *color of the
terminal's name* in a room listing (purple both-ways / blue arrival /
red departure / grey out-of-service), via a `getPresentationMml`
override that wraps the name in a `<color>` tag (see
[message-rendering.md](./message-rendering.md)). The colour carries the
meaning; the long description spells the same condition out in words
(the non-colour-alone channel) on a deliberate `look`.

**The credential** is now a `travel` **record** held in a
`CredentialWalletMixin` holder (the unified credential substrate — see
[credential.md](./credential.md)): a registered-node set plus the
register/authorize surface. The holder is base-agnostic, so it composes
around two bases — the Thing/Idea symmetry of the three-base capability
model (see [augmentation.md](./augmentation.md)):

- a carryable **`TravelCard`** `Thing` (`/domain/common/tpa/TravelCard` =
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
  routes, and a "not yet registered" note (read off the viewer's
  **identity** wallet) for destinations they haven't unlocked.
- **Destinations are named by covering Locality (D13).** The board shows
  each destination's covering-`Locality` name via
  `FastTravel.getDestinationLabel()` — resolve the node's arrival room's
  address (`AddressApi.resolveLocalityFor`) → `Locality.getName()`, falling
  back to the terminal's own presentation when no Locality covers it (the
  `LoungeTerminal` overrides it to "The Lounge" because the Warren host
  carries no stable address). Display-only: **keyword targeting stays on the
  terminal's authored `keywords`** (`teleport terminus`/`crossroads`/`lounge`),
  not the multi-word Locality name. The three board Localities (Terminus /
  The Lounge / The Last Counted Mile) are seeded under `seeds/lib/address/`
  so `AddressRegistry.postRegister` warms their prefixes at boot.
- **Selection** is node state: `selectedDestinationRef` (defaults to the
  first route). A keyword sets it; the timetable can flip it.

## The verbs

**`teleport` / `tp`** (`TeleportController`, `cmd/author/teleport.yaml`)
— **dual-mode, one verb, two forks chosen by privilege.** Only
`requiresAnimate` is verb-level; all TPA-specific gating lives *inside*
the fork so it never blocks the privileged path:

- **Self-powered** (author/developer): teleport yourself anywhere,
  destination resolved via MQL; `--target <obj>` moves something else
  instead (access-gated `teleport` / `force-teleport` actions). Subsumes
  the old object-relocation `teleport` and `goto`. Reuses `Mobile.teleport`
  with a raw-move / `forceMove` fallback and the `canTeleport` witness veto.
- **TPA ride** (unprivileged): rides the network from the node the actor
  can reach (the MQL `reachable` pool for the node). Checks the
  **instrument gate** (any reachable travel holder) → departure-capable →
  **out-of-service** (`getStatus() !== "operational"` → refuse) → keyword
  selection → **clearance** (the identity holder's `isRegistered`, read
  off the actor's own `getHostedUpdates()`) → **fare settlement** (paid
  routes only, below) → travels
  to the destination node's `getArrivalRoom()` via `Mobile.teleport`. The
  instrument gate and the clearance read are **two separate resolvers** (see
  **Identity-bound clearance** above), never one scan.

**`register`** (`RegisterController`, `cmd/movement/register.yaml`) —
records the terminal here onto your credential so you can `teleport` to
it later. **Contributed by the terminal itself**
(`FastTravelMixin.commandContributions` on the `environment` bucket), so
it surfaces only when you're at a node; the afforded node is
`context.commandSource` (validator `mustBeAtFastTravelNode`). Gated by
`requiresAnimate` + `requiresTravelCredential` + `mustBeAtFastTravelNode`.
Arrival-capable nodes only.

## Unlock = scan-to-register

You reach a node **by other means first** (walk, story), `register` your
credential at it to add it to your allowed set, *then* you can fast-TP to
it. Explicit, diegetic — not auto-unlock. The TPA fork checks
`cred.isRegistered(ref)` before travelling and refuses with "reach it
another way and `register` first".

**Born-with floor:** every credential is born registered for the
**three-node set** `BORN_WITH_TRAVEL_NODES` (`lib/credential/Credential.ts`)
— the **Terminus arrival node** (`/domain/terminus/terminal/arrival-terminal`),
**the lounge** (`/domain/lounge/terminal`), and **the paid destination**
(`/domain/newbie-wilds/crossroads/terminal`) — the documented exception to
"reach-before-travel". The hub has no foot path to the rest of the world, so
the interchange and the social hub are universally reachable by design:
onboarding's lounge→campus hop lands at Terminus (walk across the avenue to
campus), and the lounge floor node makes Gate A's free return to Dave's Bar
work even for a fresh player who never explicitly registered it. The paid
destination is on the floor because the **fare**, not registration, gates it.
The floor is preserved across hydration (the `TravelCredential` record's
`fromData` unions saved entries on top of it, never clearing).

## The transit-fare economy

A route carries an author-set **`fee`** (`TravelRoute.fee`, minor units;
`0` = free — the lounge↔terminus hops stay free). A paid ride **settles the
fare before travelling** (`TeleportController.settleFare`); insufficient
funds **refuses the ride** (no partial charge, traveller not moved). The
fare **splits two operating budgets** over banking's remittance-split seam
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
  Authority Business** (`fasttravel.tpaBusinessPath`, a minimal
  Business seed at `/domain/terminus/terminal/tpa`: proprietor-absent
  public-infrastructure operator, empty `operatingLocations` so it never
  collides with the per-terminal fare operators, resolved by path). The
  fee lands on its operating account at its authored `banksAt` (the TPA
  is **not the state** — only the treasury banks at the CB, so it banks
  commercially; every account names a real custodian, see
  [banking.md](./banking.md)). The legacy raw `tpa` accumulator row is
  re-owned to this Business by the banking boot restamp. Staff /
  appointments (a governed TPA) are the everything-is-a-business
  follow-on. The fee is a **flat base + a percentage** —
  `min(fee, base + floor(fee × rate))`, tunable via
  `fasttravel.networkFeeBase` / `fasttravel.networkFeeRate` — the
  payment-processor shape, so the TPA **collects a non-zero fee on every
  paid ride** (a pure percentage floors to zero on micro-fares). An
  unseeded world (no TPA Business resolvable) waives the levy with a
  warning rather than blocking the ride.

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

**Terminus** (`/domain/terminus/`, its own branch = the **Terminus
municipality** owner sphere, distinct from the EU campus) is the network's
central interchange, **on University Avenue across the street from the
university gate** (a cross-branch `across` exit to the EU plaza; the
standalone University Avenue terminal is **retired**, Terminus absorbs the
campus-arrival role). Its first zone is the terminal building
(`/domain/terminus/terminal`): a station **hall**, an **arrival gate**
(directionality `arrival` — the floor node the lounge routes to), **three
departure gates** (**A operational** with the free lounge return + the paid
crossroads line; **B and C out of service**), and a **ticket office**. It is
the first content to exercise the `directionality` feature (every prior
terminal was `both`) and the **`status` out-of-service** seam — the ride
fork reads `getStatus()` at authorization and refuses a non-operational
departure (**D8**; the seam stays authored-static — no dynamic breakdown).

The four terminals are **not** boot-manifest entries — they load by
**`populates:` cascade** from the single lounge root (`bootstrap.ts` holds
one hub anchor, not one entry per terminal). The three departure terminals
are never route targets, so a plain route-target cascade would never reach
them; instead each departure gate room declares `populates:` its own
terminal fixture, so standing up the hub materializes every gate + its
terminal. The office's **terminal clerk** (`TicketClerk`,
"Tootie") is the city-budget `Business`'s paid employee — she **procures a
free replacement `TravelCard`** (`procure card`, the `tpa/` command
category; a card is an instrument, no fare) and receives **wages** from the
city budget at her shift boundary (the roster tick; a bounded shift so the
wage settles). `settleShiftWageImpl` gained a **worker-account guard** that
auto-provisions an NPC worker's account so the wage lands. The **paid
destination** is the newbie-wilds **crossroads** ("The Last Counted Mile",
`/domain/newbie-wilds/crossroads`) — a minimal v0 (one designed landing room
+ one `both` terminal, free return), the integration anchor the deferred
newbie-wilds build extends.

New **committed non-guest** players receive a small **onboarding coin**
(`issueCash` at the char-gen commit; hard coin, no account, anti-farm ~20,
guests excluded) — spendable at the bar and the fare (both accept cash).

> **Reseed note (R1).** `SeederManager` is insert-only, so on an *existing*
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

## Persistence caveat (v1)

Credential registration is **session-durable, not cross-restart**:
`Avatar.save()` persists only the avatar's own fields (no inventory
persist-back; the hosted `CredentialWalletUpdate` is re-cloned each session
by `installDefaultLoadout`, so the `travel` record's registered set resets
to the born-with floor each login), so it does not survive a server restart
yet. Cross-restart durability rides future persistence work (aug-state
colocation; inventory persist-back) — see [persistence](./persistence.md).

## Seams & deferred surface

Captured in `docs/slates/tails/fast-travel-slate.md`:

- **Living infrastructure** — the `status` seam is inert in v1; the
  designed dynamism (terminals break down, routes get disrupted, an
  Authority wear/maintenance loop) is a later wave. The Authority owns
  *standard + health*, never topology.
- **Cross-restart credential durability** (above).
- Richer targeting, fares/access tiers, and disruption events.

Two things found by driving the Hinkley Hills commute route
(2026-08-01), both **behaviour, not bugs**, and both worth knowing:

- **Nobody actually sees a departures board.** The verb help says a bare
  `teleport` reads one, and neither caller reaches it: a **wizard**
  self-powers past the TPA fork entirely, and an **unregistered**
  traveller is stopped by the registration gate before the board renders.
  So the board is currently only visible to a player who is already
  registered for the selected route — which is the audience least in need
  of it. Whether an unregistered traveller should be shown the routes
  they cannot yet take (a *timetable* is public; a *ticket* is not) is a
  design question this doc does not answer.
- **⚠ A stale terminal row silently deletes a route, and the refusal
  lies.** `SeederManager` is insert-only, so adding a route to a terminal
  that already has a row in `domain` leaves the live network on the old
  list. `teleport <kw>` then answers **"no route here goes to '<kw>'"** —
  which is *also* what an unauthored route says. The two are
  indistinguishable in a transcript, and the good outcome (the
  registration gate) is the one that proves the route is really wired.
  When a route is added to an existing terminal, the row has to be
  re-seeded — content-pack reconcile does this, a bare dev restart does
  not.
