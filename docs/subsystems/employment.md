# Employment

The **employment engine** — a real, actor-agnostic model of jobs, shifts,
wages, and tips, replacing the NPC-only placeholder at Dave's Bar
(forced-teleport "shifts" + manual operator wages + intrinsic-`Crafter`
capability faked by location). Lives in `lib/employment/` with the gated
`EmploymentApi`/`EmploymentLogic` pair. NPCs are the v1 consumer; players
are supported at the *relationship* layer for free and blocked only at the
*capability* layer by a named, deferred seam.

## ⭐ The governing split — Business models the economy; an organization models the chart

> **Business models participation in the economy. An organization models
> the chart.** Business carried both, and that was the conflation.

The org chart — **positions**, **holders**, and the **appointing
authority** — lives on `OrganizationMixin` (`lib/employment/Organization.ts`),
which answers *who holds position P in organization O?* and its inverse
*what does actor A hold, anywhere?* identically for a ministry, a shop and
a newspaper. `BusinessMixin` **requires it on its base** and adds only what
trades: `banksAt`, `operatingLocations`, the P&L account path, wage
settlement.

Before the split, content that needed a position had to stand up a
*Business* — a thing that does not trade, pretending to. Terminus's
Registry keeps records and has a Magistrate's seat; it was a Business
purely because that is where positions lived. `obj/Organization.ts` is
what such content names now.

**The seat/staff line, expressible for the first time:**

| | prescribed by law? | is |
|---|---|---|
| Compact **Office** | yes | `OFFICE_APPARATUS` |
| government **seat** | yes | a `Government.seats` entry **pointing at** a position |
| **personal staff** | **no — serves at pleasure** | **a position nobody points at** |

⭐ **The difference is whether a constitutional document points at the
position. Same substrate; the pointer is the law.** So a Communications
Director is a *position*, never an Office — nothing in a constitutional
document points at it.

## The governing decision — Business is its own entity

The thing that owns the proprietor, the positions, the roster, the account,
and the operating locations is a dedicated **`Business` `Idea`** — *not* a
mixin on the venue `Location`. The Bar stays a dumb `Location` with **zero**
employment data. This is what lets a job span locations, a proprietor be
absent, and (the clincher) the business **outlive its proprietor** — the
Dave→Augie→Mara succession the cast implies needs the business standalone
with the proprietor as a replaceable edge.

Routing falls out cleanly: order/tip resolution finds the **present agent
with an active `Maker` capability** — it never consults the room about a
business. The house **account keys on the Business's own path**
(`getAccountPath()` = its `templatePath`), so order income and shift wages
settle on one P&L account that survives a venue move. Reverse lookups (room
→ business) go through the `EmploymentLogic` **business index**, never a
field on the room.

## Data model (`lib/employment/`)

Four value objects + two mixins + the concrete entity:

- **`Position`** — a job's terms: `{ key, label, noun?, wageRate /* minor
  units per game-hour */, fulfills? /* disciplines served */, headcount?,
  requires?, purchases?, reportsTo? }`.
  ⭐ **`noun` is what ONE holder is called** — `bartender`, `clerk` — and
  it feeds the handle chain's second rung (`getPositionNoun()`), so an
  NPC in a job is *"a baker"* with nobody typing it onto the NPC, and
  **stops being one the day they are dismissed**. ⚠ It is a second field
  because `label` cannot supply it: every shipped label is a gerund
  (*"tending bar"*, *"on the road"*) and there is no honest
  gerund-to-noun transform. See [presentation.md](./presentation.md). The
  `Money`/`Charge` precedent (data + `serialize`/`fromData`).
  ⭐⭐ **What a seat GRANTS is data on the seat, never a marker mixin** —
  `purchases` (spend the house's money) and `fulfills` (serve an `order`
  here). See *Capability grant* below for why that replaced a
  `confers: string[]` list of mixin names. ⚠ **`wageRate` stays on
  `Position`**: compensation attaches to the *position*, which is what
  makes the roster's `{positionKey, assignee}` shape right rather than
  incidental. **`purchases?: boolean`** (libations) — the position buys
  for the house: its holder may `wallet use house`, and a non-Avatar
  holder is dealt the **house card** at hire (below). A data field, never
  a marker mixin — nothing else on `Position` is authority-shaped.
- **`ParLine`** (`lib/employment/ParLine.ts`) — one line of a Business's
  **par manifest**: `{ category, minGrade?, level, unit: 'L'|'count'|'kg',
  supplier?, exemplar? }`. `category` matches a **material tag** (a
  bottle of gin, a bag of ice, a crate of limes — the bulk interior
  material for `L`/`kg`, `Tangible.getMaterial()` for `count`) or a pool
  glass's own `getCategory()`; `supplier` is a Business template path.
  ⭐ `exemplar` is the template the house ORDERS when the line is short
  and there is none left to point at — without it a venue that opens
  with an empty shelf can never order anything and never opens at all
  (logistics). The par is the
  owner's policy; what is *on hand* is a derived read (below).
- **`Authority`** — the `PrincipalRef` tagged union: *who may fill this
  organization's positions?* See below.
- **`Employment`** — one actor's relationship to one **organization**:
  `{ organizationPath, positionKey, status, hiredAt, onShiftSince }`.
  Immutable value object (`withStatus` returns a copy). `EmploymentStatus`
  vocabulary = `employed | on-shift | off-shift | quit | fired` (+
  `EMPLOYMENT_STATUSES` validation array). One holder relation for
  everything: a volunteer is a **wage-0 employee**, wage and shifts stay
  optional, and there is no consent model on top. ⚠ `fromData` still reads
  the legacy `businessPath` key, so records stored before the split hydrate
  with no migration — paths did not move, only the name.
- **`Roster`** — a Business's schedule: ordered `RosterAssignment
  { positionKey, assignee /* templatePath */, schedule: ShiftEntry[] }`;
  `ShiftEntry { days: number[], hours: [start, end) }`. `evaluate(assignment,
  date)` is the pure day/hour-window match lifted verbatim from what the
  `shifts` brain read inline before this build.
- **`OrganizationMixin`** (`lib/employment/Organization.ts`, marker
  `_mixinName='OrganizationMixin'`) — the chart. Persistent fields
  `appointingAuthority`, `proprietorPath` (the legacy hydration slot),
  `positions`, `rosterSlots`, `parentOrganization`; the reads
  (`getPositions`/`getRoster`/`getReportingChain`/…) and the **seven holder
  transitions** `hire` / `endEmployment` / `ensureRostered` / `beginShift`
  / `endShift` / `beginCover` / `endCover`, gated
  `AnyOf(SelfOnly, FromTemplate('/platform/idea/api/employment'))`. Every record
  write is keyed on the organization's own path — which *is* the
  participant contract below. `obj/Organization.ts` is the concrete
  instanceable non-trading organization.
- **`Business`** — `BusinessMixin` (marker `_mixinName='BusinessMixin'`) +
  the concrete **default-export `BusinessEntity`** (`class BusinessEntity
  extends BusinessMixin(PostRegistrationMixin(Idea))`). The concrete class
  name differs from the `Business` **interface** + `BusinessMixin` on purpose
  (the `Bank`→`BankCounter` convention — a same-named class+interface+mixin
  triad recurses as a base type). It **requires `OrganizationMixin` on its
  base** — a compile-time constraint, the `MountableMixin`-requires-`Slotted`
  idiom, rather than a nested factory: TypeScript loses an anonymous mixin
  base's members through a nested generic mixin, which would make
  `positions` and `getProprietor` invisible on every concrete Business.
  Its own persistent fields are `['operatingLocations', 'banksAt']`; the
  chart's are on the mixin below it. `canDestruct()` refuses (seeded
  singleton-style).
- **`EmployedMixin`** — on `Character` (actor-agnostic; sparse null-default
  `employments` field — the `BeliefStore`/`Status` precedent, so an
  unemployed Character carries nothing). Pure storage + the derived
  conferral read. The privileged mutators (`_setEmploymentStatus` /
  `_upsertEmployment` / `_removeEmployment`) carry a **participant
  contract** — the caller must be the **organization party to the record**
  (`FromMixin(Mixins.Organization)` + a relational `where` requiring the
  written record key to be the calling organization's own path), with a
  narrow `FromTemplate('/platform/idea/api/employment')` janitorial arm (lazy
  standup means a `quit` can outlive its organization's live Idea). The
  transitions themselves live on `OrganizationMixin` — so the
  organization acts on its own employee records and the engine keeps
  orchestration (roster evaluation, wage settlement, the clock).
  `isFulfilling(discipline?)` is the *am I the on-duty hand here* read —
  three conditions, all of them (below). `getActiveEmployments()` is the
  *what does A hold, anywhere?* read. `clockOn`/`clockOff` are the
  actor's own shift acts (below).

## ⭐ The appointing authority

`lib/employment/Authority.ts`. `proprietorPath` was a templatePath — a
**specific entity** — so authority under it could never be handed off,
exactly the defect the check-the-office doctrine exists to prevent. It
becomes a tagged `PrincipalRef`:

| kind | resolves through | founder passes? |
|---|---|---|
| `entity` | templatePath match — the shipped proprietor edge | no |
| `office` | `CompactApi.holdsOffice` | **yes — founder default** |
| `seat` | `GovernmentApi.holdsSeat` | no |
| `committee` | `CompactApi.isCommitteeMember` | **yes — pool-of-one backstop** |

⭐ **Staff follows the seat.** Because the authority *is* the office, a
handover moves who may appoint **with no employment or roster record
touched**.

`EmploymentApi.holdsAuthority(principal, ref)` is **one function answering
one question**, dispatching on the tag and nothing else. There is
deliberately **no** "which authority does this actor satisfy?" helper —
that shape turns a refusal into a downgrade. It fails closed throughout:
an unauthored authority refuses everyone, each delegate already fails
closed with no registry, and the `committee` arm narrows with
`PlayerApi.isAvatarStuff` before the playerId-keyed group read.

⚠ **The *founder passes* column is load-bearing, not trivia.** Only
`office` and `committee` carry the Art. XI default, so **an authority the
founder cannot satisfy is one nobody can satisfy on a cold box** until a
human edits a group by hand. (The former `AccessApi.isAuthor` fallback —
an always-empty `core` group — is gone since content packs wave 3; the
operator override is the PM seat, below.)

⚠ **There is deliberately no `author` kind.** The operator axis is an
override layered *on top of* an authority (`isProprietorOf`'s
`CompactApi.holdsOffice(subject, 'prime-minister')` arm — the Prime
Minister as operator), never an appointing authority in its own right.

⚠ **No seed edits were needed.** `Authority.fromData` reads a bare string
as `{kind: 'entity', path}` — the legacy shape, same value under the new
type — and the mixin keeps `proprietorPath` as a legacy hydration slot
that `appointingAuthority` overrides when both are present.

## Nesting

`Position.reportsTo` walks *within* an organization
(`getReportingChain(positionKey)`, nearest superior first);
`parentOrganization` walks *between* them
(`org.organizationChain()`, nearest parent first). Both are
optional, so every shipped Position and Business is unchanged.

⭐ **The first content consumer is the Goodkin branch**, which names
`/corpo/goodkin` as its parent ([corpo.md](./corpo.md)): a subsidiary that
trades, inside a holding company that does not. ⚠ **Nesting is not
inheritance** — a position is held where it is authored, so the branch's
tellers stay the branch's and do not appear on the company's chart.

⚠ **Both walks refuse a cycle rather than looping** — loudly, not by
truncating: a truncated chain looks like a valid answer, and the failure a
guard actually prevents takes the process with it. A *dangling* superior
or an unresolved parent just ends the chain: that is a gap, not a
contradiction.

## The uniform reads

`organization.holdersOf(positionKey)` answers *who holds P
in O?* — durable templatePaths, identical for a ministry, a shop and a
publisher. Its inverse is the actor's own `getActiveEmployments()`.

⭐ Both, plus the publishing entitlement, read **one** module-private
`holdersByPositionImpl` scan: live non-terminal `Employment` records
unioned with the authored roster (what makes a never-ticked, lazily
stood-up organization's holder provable), with **an explicit exit
suppressing the roster entry**. That is deliberate — it means the
quit/fired suppression `holdsSeat` implements exists exactly once and
cannot drift into a second copy.

## Appointment — the `appoint` verb

`appoint <player> to <position> at <organization>` (`employment`
category), afforded universally on `Persona.commandContributions.self` for
the same reason `office` is: the gate is the authority, not the
affordance.

⚠ The gate is a **field** validator, `mustHoldAppointingAuthority`, on the
`organization` argument — not a verb-level one. `CommandContext` carries
the giver but not the bound model, so a `requiresX` validator can only ask
giver-side questions, and the authority here belongs to *the organization
the argument names*. Same gate, homed where it can see the thing it gates
on; the controller re-derives no authority.

Without this verb **no position could ever be filled by a human**, which
is the same defect as the broken `office assign`.

## ⚠ Employment is independent of money movement

`postTransaction` is the banking chokepoint and **takes no employment
argument**; nothing in it consults employment. Paying someone with no
employment record must remain *identical* to paying an employee off-book —
that is what makes paying under the table an emergent act rather than an
unmodellable one. **Do not couple them.**

## The gated Api/Logic pair

`EmploymentApi` (`api/employment.ts`, thin forwarding shell) →
`EmploymentLogic` (`platform/idea/api/EmploymentLogic.ts`, `@internal @Unshadowable
extends Idea` at `/platform/idea/api/employment`, HMR-able; every method gated
`FromModule('/api/employment#EmploymentApi')`). Surface:
`holdsAuthority` / `mayPublishAs` / `holdersOf` / `organizationChainOf` /
`isProprietorOf` / `hire` / `fire` / `quit` / `businessAt` /
`businessOfProprietor` / `beginCover` / `endCover` / `tipRecipientFor` /
`shiftStateOf` / `settleShiftWage` / `tickRoster` / `boot`.

- **Proprietor authority** = the organization's **appointing authority**,
  checked by `isProprietorOf`, with the Prime Minister's seat
  (`CompactApi.holdsOffice(subject, 'prime-minister')`) as the orthogonal
  operator override (content packs wave 3 — the former `isAuthor` arm). Not a Zone `ownerGroup` — `AccessApi` cannot
  represent an NPC owner. For an `entity` authority this is
  byte-identical to the shipped `proprietorPath` check.
- **`mayPublishAs`** lives here rather than on the press face because it
  reads the one holder-resolution path — routing it through a press logic
  singleton would mean either a thin wrapper or a cross-logic import, both
  banned. See [press.md](./press.md).
- **Business index** — businesses are found by the `BusinessMixin` marker
  (the `SlotLogic`/`LocomotionLogic` enumerate-by-scan precedent), cached on
  the **singleton instance** (a `StuffApi.clearAll` recreates the singleton
  → fresh cache; rebuilt on a miss). `businessAt(locationPath)` /
  `businessOfProprietor(subject)` read it.
- **Fixture-keyed attribution + derived lazy standup.** `operatingLocations`
  names the **fixture** a Business operates (a terminal / vending unit), not the
  room — so two venues sharing a room each resolve their *own* operator
  (`businessAt(fixture)` is a sound 1:1; two businesses claiming one fixture is
  an authoring error). `ensureOperatorAt(fixturePath)` is the async lookup that
  stands the operator up **lazily** if it isn't live yet — derived from the
  Business's own `operatingLocations` template data via a cached reverse index
  (`operatingLocation → BusinessTemplatePath`, filtered cheaply by the field's
  presence, `isBusiness`-verified after standup). This retires the per-venue
  standup hooks: **no** manifest entry, **no** `Bar.postRegister` /
  `TicketClerk` clone — the first `businessAt`-style query at a fixture (an
  order, a fare) stands the Business up. Consumers: `OrderController` (the bar)
  and `TeleportController.settleFare` (the transit fare) call `ensureOperatorAt`;
  the roster tick's live-scan then processes it.

## Roster-driven on-shift state (`runTick`, on the game clock)

`EmploymentApi.boot()` (wired in `AppBootstrap` **after** `BankingApi.boot()`
— wages call banking) runs one immediate pass then self-registers a
recurring game-time tick (`WorldClockApi.every(1 game-hour)` — freezes with a
paused world, so accrual freezes too). Each pass enumerates every Business
and, per roster assignment:

- **lazy-materializes** an `Employment` on the assignee (the roster is the
  single source of truth — the seeds carry no employment block);
- **off→on**: stamps `onShiftSince = now`;
- **on→off**: settles the shift wage off the captured record *before*
  clearing `onShiftSince`, then flips to off-shift;
- a `quit`/`fired` record is left alone (an explicit exit is never
  resurrected by the seed roster).

The pure logic lives in the private `runTick`; the gated public `tickRoster`
and the schedule callback both delegate to it (no intra-singleton gated
`this.x()` call trips the gate).

## ⭐⭐ Capability grant — what a SEAT lets you do

A position grants two things, and both are **data on the position**:

| field | what the holder may do | read |
|---|---|---|
| `purchases?: boolean` | spend the house's money (`wallet use house`) | `buysFor()` |
| `fulfills?: readonly string[]` | serve an `order` in those disciplines | `isFulfilling(discipline?)` |

`Employed.isFulfilling(discipline?)` is **three conditions, all of them,
all the time**: the actor is **on shift**, the seat its house authored
lists the discipline under `fulfills`, and the actor is standing
somewhere that house **operates** (`getOperatingLocations()` against the
room's path and its contents' identity paths — the `resolveHouse` walk).

⭐ **Employer-bounded is the point of the third leg.** An on-shift
bartender who walks into the smithy does not fulfil smithy orders.

⭐ **The discipline list is the point of the second.** A venue can run
two trades off one business — the Hearthworks runs a smith and a cook
over both its rooms — so *"serves orders here"* cannot route an order,
and nothing downstream would catch a wrong pick: a recipe's discipline is
**credited, never gated**, so the smith would cook the stew successfully
and be credited with `cooking`.

⚠ **`fulfills` says who MAY, never who DOES when several qualify.**
`CraftingLogic.resolveMaker` breaks ties on the lowest identity path,
which is predictable rather than right — two cooks in one kitchen wants a
queue. That is arbitration and belongs to the crew substrate
(crew-slate).

### ⚠ What this replaced, and why

Until the trades-and-labor build a Position carried `confers: string[]` —
mixin names, folded into the **augment** walk by a structural soft-lookup
of `getConferredMixinNames()`, so an on-shift holder's gated `MakerMixin`
went active. Three things were wrong with it:

1. **Augment gating is for physical implants and innate gifts**, not for
   a means test. A job is not a prosthetic.
2. ⭐⭐ **The grant was unreachable by a player.** `MakerMixin` was
   composed on exactly one class — `Crafter`, an NPC class — so a player
   who took the bartender's seat received *nothing*. A capability nobody
   can compose is a capability no player can ever hold, which is why the
   labor market had nothing to give anybody.
3. `confers` was open-ended: a row could name any mixin at all.

`MakerMixin`, `MixinApi.isMaker`, `Mixins.Maker` and `Crafter` are gone;
the seven `Crafter` rows are plain `Cast`. ⭐ The augment fold itself
**stays** — `Shade` overrides `getConferredMixinNames()` to confer
`AetherMixin` intrinsically (attuned with no implant and no slot, and
species `innateMixins` is shared reference data a shade cannot write),
which is a genuine augment conferral and exactly what the seam is for.
What was deleted is `EmployedMixin`'s implementation of it. See
[augmentation.md](./augmentation.md).

## ⭐⭐ The opening, the criterion, and the sign

An **opening** is `headcount − holders`, **derived, never stored** — so
nothing can decrement it wrong: a hire closes it, a `quit` reopens it, a
`vacated` reopens it, by arithmetic. `headcount` absent = no opening is
ever advertised, which is every seat that does not author it.

`Position.requires` is what the seat asks of an applicant, and ⛔ **the
vocabulary is CLOSED**: `{ gigs, discipline, band }`. A hiring criterion
judges a PERSON, so lens 6's governance limb applies — name the criterion
and name the appeal — and every criterion in the vocabulary is something
a player can go and DO. Nothing may select on species, lineage, trait,
renown or wealth, and `Position.fromData` **throws** on an unknown key
rather than coercing it away (a silently-dropped criterion is a sign that
lies and a refusal that never fires). `lint:openings` catches it first.

- `Organization.openingsFor(key)` / `openings()` — the derived reads.
- `Organization.considerApplicant(applicant, key)` → `ApplicationVerdict`,
  which carries **`wanted` and `held`** on every refusal. Ordered
  cheapest-lift-first: `already-held` → `no-opening` → gigs → band.
- `ContractApi.settledGigsBy(identityPath)` — completed work, keyed on the
  durable identity path.
- `EmploymentApi.noticesAt(here)` — every opening advertised where `here`
  is, over `operatorsAt` (the one candidate walk, shared with
  `BankingControllerBase.resolveHouse`). ⚠ **Live businesses only**:
  standing a house up is an economic act and walking into a room must not
  be one. The authoring rule that makes that honest — an advertising
  house is a `boot:` producer of its own pack — is `lint:openings`'s.

**The sign has no host, no mixin and no object.** `LookController` prints
one line per opening, so a venue with an open seat *cannot fail to
advertise*: the notice comes off the same arithmetic that decides the seat
is open at all. ⚠ It rides its **own uncarded scene**, not the room body —
a body handed to `CardApi.open` is suppressed from the transcript in
favour of a card that never renders it, which made the sign invisible in a
browser until the live drive found it. See
carded-prose-slate.

## `apply` and `clock` — the player moves first

- **`apply [for <position>] [at <organization>]`** — the applicant asks;
  the organization decides. ⭐ Standing is conferred by the employer.
  Every refusal names the number and what lifts it.
- **`clock on` / `clock off`** — ⭐⭐ a shift is something you CHOOSE to
  start. Holding a job puts you on the chart, not on the clock.
  `beginShift` runs from exactly one place, the roster tick over authored
  `rosterSlots`, so a player who applies holds a seat with no roster
  entry: without `clock` they would never start, never be paid and never
  be granted anything. The alternative — writing the applicant a slot
  with the seat's hours — pays them present or not, which is the AFK wage
  lens 6 names a failure. A recorded lean (`livelihood-slate` §5.4:
  *voluntary clock-in, employer-bounded*). `clockOff` settles the wage
  through the roster tick's own off-transition, invoked by the holder.
  ⚠ A rostered NPC is untouched: the tick iterates assignments and an
  applicant has none.

## Wage settlement at shift-end

The wage is a **lump paid once, at the on→off transition** — `wageRate ×
(offTime − onShiftSince)` game-hours — not a continuous sweep (the *shift* is
the settlement unit, as *completion* is the gig's). Rides the roster tick's
off-transition, so there is no separate wage schedule; a deliberate
clock-out is an early off-transition → pays the partial; a paused clock
accrues nothing. `settleShiftWage` **skips the proprietor's own cover**
(no self-wage) and pays from `business.getAccountPath()` via
`BankingApi.ensureVenueAccount` + `payWage`. `house payroll` stays the
operator override (a one-off bonus), now redundant for the normal loop.

`OrderController` income **and** `HouseController` pnl/payroll re-key to the
Business account (`EmploymentApi.businessAt(location)?.getAccountPath()`), so
income and wages roll up into one deficit P&L.

**Worker-account guard.** `settleShiftWageImpl` provisions the worker's
account if absent (`ensureVenueAccount(employeeKey, employeeKey, '')`) before
`payWage` — NPC workers never opened one, and `payWage` throws without it.
Additive + general; closes the same gap in the bar loop.

**Second employment consumer — the Terminus city budget.** Terminus's
**municipal city-budget `Business`** (proprietor-absent, `/world/terminus/
budget`) is the second consumer after Dave's Bar: it lists the operational
departure gate in `operatingLocations` (so the transit fare's city share
attributes to it, un-spoofably), pays the **terminal clerk** (a bounded roster
shift so the wage settles at the boundary), and closes the conserved fare-in →
wage-out loop. The budget Business stands up **lazily** — derived from its own
`operatingLocations`, on the first `ensureOperatorAt(fixture)` query (a fare at
the gate) — no `TicketClerk`/`Bar.postRegister` clone. See
[fasttravel.md](./fasttravel.md) § Terminus.

## The `shifts` + `covers` brains

Presence is a **consequence** of employment state, kept in brains so it
stays hot-swappable:

- **`shifts`** — reads `host.shiftState()` (sync) and
  teleports on-shift → `config.behindBar`, off-shift → `config.offstage`.
  The game-clock schedule match is **gone** (the schedule lives on the
  Business roster now). Not presence-gated (off-stage cast must move out
  before a player arrives). config: `{ behindBar, offstage, railStool? }` —
  `railStool` is a reserved key for the deferred off-shift-at-the-rail
  presence (v1 presence is binary). `config.offstage` names the venue's
  own **`Offstage`** row (below); both shipped venues park through it.
- **`Offstage`** — the off-shift parking role (content packs wave 4b,
  graduated out of the lounge): `OffstageMixin` in `lib/employment/`
  (a marker + the one invariant — never `Exitable`; `Mixins.Offstage`,
  `MixinApi.isOffstage`) and the clonable `platform/location/Offstage`
  (singleton per template path, Visible/Detailed for the operator who
  teleports in) that every venue's `offstage` row names —
  `/world/lounge/location/offstage`, `/world/terminus/hearthworks/location/offstage`.
  The world conserves identity: an off-duty NPC is relocated, never
  destroyed and respawned, so each venue with a scheduled cast needs
  somewhere for that cast to *be*. Materialized on demand by `shifts`
  (`StuffApi.singletonOrClone`). The hearthworks roster is 24/7, so its
  parking never fires in shipped hours; the room exists so a shortened
  schedule parks Berta and Odo somewhere rather than nowhere. Tests:
  `lib/employment/__tests__/Offstage.test.ts` (two venues, no bleed)
  and one per venue beside its content (`world/<venue>/__tests__/offstage.test.ts`).
- **`covers`** — the proprietor covers gaps. On a presence-gated cadence, if
  **no other active on-shift maker is present** in the proprietor's location,
  `self.beginCovering(business)` upserts a **transient, on-shift**
  Employment against the first **`fulfills`** Position (falling back to
  `positions[0]`) — reusing the whole on-shift path, so the covering
  proprietor serves an `order` and a customer still finds a fulfiller.
  ⭐ The seat a cover covers is a fulfilling one: a proprietor steps behind
  the bar to serve, not into the bookkeeping. Unpaid by construction (the wage settlement skips a
  proprietor-held Employment, and the tick never governs the proprietor).
  `endCover` drops it when a real bartender is back. v1 = clause-unheld only
  (demand has no measure yet); `beginCover` does **not** verify
  proprietorship — the brain gates on `businessOfProprietor`.
- **`restocks`** (libations) — the keeper reads the par sheet and buys
  the shortfall; **`consigns`** — a producer's floor hand carries stock
  to a distributor's counter and consigns it as the business. Both are
  kernel, generic, and drive the **literal verbs** through
  `CommandApi.forceCommand` (`wallet use house`, `buy`, `put … on`,
  `pour … into`, `get`, `wash`; `get`, `consign … --ask`) — nothing an NPC
  does is unavailable to a player. See [behavior.md](./behavior.md) and
  [retail.md](./retail.md).

## ⭐ The house account in the wallet, and the par manifest (libations)

> **The wallet's active account is the principal you trade as.**

The one kernel gap the bar's supply chain needed was *a purchase paid
from a business account*. No custodial shape was needed: a
`PaymentCredential` is a set of linked account ids and an active pointer,
and settlement never checks that the payer *owns* the routing account
(it checks `authorize` and balance only). So the conferral is a **link**:

- **`actor.buysFor(): Promise<BusinessStuff[]>`** — every
  Business where the actor holds a non-exited `purchases` position, plus
  the one whose proprietor they are.
- **`wallet use house`** (`WalletController`) — resolves the house via
  `buysFor`: the business operating **here** (`businessAt`), else the
  single one, else `ambiguous-house` (names them); links its operating
  account into the holder's credential and makes it active
  (`not-staff` / `no-venue-account` otherwise). Bare `wallet` names a
  house account as such ("the house account of Dave's Bar").
- While active: **`buy` stamps the chattel to the business**
  (`BuyController` reads `receipt.accountId` → `BankingApi.ownerKeyOf`
  → a live `Business` → `ChattelApi.stamp(item, business)`, the
  `organization` owner arm; "for Dave's Bar" in the scene); **`consign`
  consigns as the business** (the listing's `consignorKey` is the
  business path, its operating account is the payout; an unstamped good
  the hand holds is stamped to the business first). `house stock` /
  `house par` act on that business.
- **`quit`** (`platform/idea/cmd/employment/QuitController.ts`, `quit
  [<business>]`) is the player's way out; `actor.quitJob` / `org.dismiss`
  unlink the operating account (`BankingApi.unlinkAccount`), and a relog
  re-clones the wallet app, so the link never outlives the position.
  Both are async now.
- **The house card** — NPCs run no `installDefaultLoadout` (no implant,
  no wallet app), so the NPC shape of the same conferral is a
  **`PaymentCard`** linked to the operating account, dealt into the
  holder's inventory at `hire` / roster materialization when the
  position has `purchases: true` and the holder is not an Avatar
  (`issueHouseCardImpl`, idempotent, `UNCAPPED` — seat authority, not a
  number). Same credential kind, same `wallet use house`, same `buy`.
  Players get nothing minted.

**`house`** (`HouseController`) no longer carries `requiresWizard` — that
is the TypeScript code-trust axis and nothing else; venue authority is
the SEAT (`resolveHouse`: the business here if the giver `holdsPosition`
or `isProprietorOf`, else `buysFor(giver)[0]`, else `not-staff`).
`house par <category> <level> [--grade <band>] [--from <business>]` edits
the manifest (unit from the suffix — `6L`, `5kg`, bare = count; `0`
strikes the line). `house stock` emits the sheet as prose and shows the
live `stock` card on a screen ([display.md](./display.md); `no-display`
without one).

**`business.stockSheetFor(viewer): StockSheetLine[]`**
(`{ line, onHand, shortfall }`) — for each par line, the on-hand total
over the goods the **viewer perceives** from where they stand
(inventory + location, descending open containers — a bottle in a rack,
a lime in a crate — never sealed ones, filtered by
`PerceptionApi.perceives`). One function, two consumers: the controller
and the `restocks` brain. `goodsFor(viewer, category)` returns the
matching items so a buyer can name what to `buy`.

⭐⭐ **The matcher itself lives in one place** —
`lib/employment/CategoryMeasure` — and answers both *does this count
against the category* and *how much of the unit is it*. The sheet, the
keeper's exemplar scan, the carter's crate finder and a `supply`
contract's tally all read it. ⚠ There were FOUR copies of that question
and two were wrong in different ways: one read a bottle's **vessel
kind** as its par category (so no bulk line ever matched), and one bound
a template path (so *eight of that exact row* was the only way to
satisfy an order for six litres). A business is denominated in category
and unit; everything it handles is denominated in objects; the
translation is where it kept going wrong.

`Business.parLines` (persistent + authorable; `getParLines` /
`setParLine` / `removeParLine`). The lounge's `business.yaml` authors 46
lines (the vinegar line joined with the fermentation build — the cook
buys the failure path), every bought line
`supplier: /world/terminus/counting-houses/distributor/idea/business` (the distributor's own
pack since fermentation D10);
glassware lines carry no supplier (nobody consigns glasses yet, so
breakage shows as shortfall the brain cannot buy back).

### ⚠⚠ A supplier is found by the ROOM its counter stands in

A par line names a `supplier`, and `restocks` turns that path into a
place a hauler can go and buy: `counterRoomOf` walks the supplier
business's `operatingLocations` for one that **holds a `Stock` or a
`ConsignmentShelf`**, and that room becomes the order's `--from`. A
counter FIXTURE is not that room — its own contents are its goods — so a
house that lists only its counter is not a supplier at all, and the
whole bucket is dropped with a `continue`.

⭐ **The failure is completely silent.** The brain declines nothing (it
never reaches `job post`), the dispatch log prints declines only, and the
row suite asserts rows. The trades-and-labor build lost the tailor's
cloth order — and with it its own *work from more than one supplier*
acceptance criterion — to exactly this, and found it only by reading
`contracts` out of Mongo on a live world.

⚠ So **list the room as well as the fixture**; the cash-and-carry's row
is the shipped precedent (`.../distributor/thing/counter` *and*
`.../counting-houses/cash-and-carry`). `lint:openings` arm 5 holds it:
every `parLines[].supplier` must resolve to a house whose operating
locations include a room something props a counter into.

**The player path to the seat** is diegetic: Dave's tree-dialogue
(*"Looking for work?"*) carries a `dispatch` effect that runs `appoint
$player to keeper at /world/lounge/idea/business` as Dave, guarded on
`position:/world/lounge/idea/business eq false`
([npc-dialogue.md](./npc-dialogue.md)). No wizard anywhere in the loop.

## Tips — the tip jar (`tip` / `collect`)

Tips are **physical cash**, two routes, never the bar's P&L:

- **`TipJar`** — a `Container` `Thing` fixture (`platform/thing/TipJar` —
  a commons class, a jar is a jar) that holds `Coin`, `Detailed` so it
  fills visibly. Affords `tip` + `collect` from the environment bucket
  (the `Menu` affordance pattern). The *template* is the hospitality
  trade's (`/trade/hospitality/thing/tip-jar`); the lounge's bar
  `props:` an instance onto its back-bar.
- **`tip <amount> [--eft]`** — **cash** (default): `BankingApi.settle` cash
  moves coin patron→jar, off every ledger (anonymous, the under-the-counter
  take). **EFT** (`--eft`, or the automatic fallback when the patron lacks
  coin): `BankingApi.transfer` patron→the on-shift server's account —
  recorded on the *server's* ledger, the bar account untouched. The patron
  is never blocked; the cash/EFT choice is the anonymous-vs-recorded story.
- **`collect`** — the on-shift bartender scoops the whole jar into their
  holdings, gated on `Employed.isFulfilling()` (on-shift-aware). Per-shift
  attribution falls out — whoever's on shift empties it. NPC auto-collect at
  shift-end is a deferred beat.
- **`tipRecipientFor(patron)`** — the present on-shift server (the
  `resolveMaker` scan, one cardinality across), shared by the EFT route and
  `collect`'s gate.

## The Business stands up lazily (derived, no hook)

⭐ **An unvisited venue mints nothing.** Wage settlement runs only for
stood-up businesses, so a ghost venue causes no wage inflation. Lazy
standup is economically load-bearing, not a performance nicety — and it
is residency's symmetric partner: fault in on demand, evict the cold
tail (graduated from the content-packs slate, 2026-09).

A Business is **not** stood up by a `postRegister` hook (the old
`Bar.postRegister` / `TicketClerk` clones are gone) nor a manifest entry. It
stands up **lazily**, derived from its own `operatingLocations`, on the first
`ensureOperatorAt(fixture)` query — an order at the bar, a fare at a terminal
(see **Fixture-keyed attribution + derived lazy standup** above). Idempotent
(`singletonOrClone`); the roster tick's live-scan processes it once live. A
context with no Business template for a fixture resolves to null and falls
back gracefully. The **city budget** (Terminus) and the **frontier settlement
budget** (the newbie-wilds crossroads) both stand up this way — each names its
own terminal fixture in `operatingLocations`.

## Fixture resolution goes through MQL

The affording-fixture resolvers enumerate the room via the MQL **`peers`
seed** and filter by **type/capability**, not a hand-rolled containment
scan: `TipJar.resolveIn` / `Menu.resolveIn` statics
(`MqlApi.resolveMany('peers').stuff.find(instanceof …)`, commandSource
affordance fast-path kept) and `BankingControllerBase.resolveBank` /
`presentBartender` (`.find(isBank / isFulfilling)`). Matching by type means a
honey jar on the same back-bar can't win on the `jar` keyword; the
`instanceof`/capability filter is the interim type check a future MQL
type-predicate subsumes. `resolveVenue` stays `context.location` (the room
the actor stands in), not a scan.

## Cast (`domain/lounge/`)

Dave → pure **proprietor** (the `proprietorPath` edge on the Business seed;
`covers` brain, no `shifts` schedule). The four staff (Mara/Remy/Sloane/
Augie) → roster **assignees** (schedules lifted verbatim from the old NPC
seeds, incl. Sloane's midnight-wrap two-window shift); each keeps
`class: /platform/agent/Cast` — ⭐ plain `Cast` since the maker marker
retired; what lets the one on shift serve is the house's `fulfills` SEAT —
drops its
`shifts` schedule, carries no employment block (materialized by the tick).
The Business seed (`/world/lounge/idea/business`) authors the `bartender`
Position (`fulfills: [bartending]`, `wageRate` a tuning placeholder), the
roster, and `operatingLocations: [/world/lounge/location/bar]`. Positions
are rows on a Business — there is no industry-level position artifact,
so the hospitality trade ships none and the venue keeps them.

## Compensation bases (the arrangement generalization)

A `Position` now carries an optional **compensation term** —
`PositionData.compensation?: { basis, rate?, share? }`
(`lib/employment/Compensation.ts`, `COMP_BASES`) — the work-contracts
build's arrangement schema landing: every business model is authored as
**terms on the arrangement, never a new subsystem**. The four-basis
model:

1. **time** — pay for *holding a maintain clause* (the shipped wage:
   `wageRate × shift-hours` at the on→off boundary; the employer bears
   demand risk). **The default reading of an absent `compensation`** —
   `Position.fromData` never materializes a default, so legacy seed
   blobs round-trip byte-identically and the shipped time-wage path is
   untouched except one guard: a non-time basis accrues **no** shift
   wage (pinned by `employment-wages` / `bar-loop` /
   `city-budget-wage` and the in-file `comp-bases` regression).
2. **per-settlement** (piece-rate — the miner's basis; the worker bears
   demand risk): `EmploymentApi.settlePiecework(business, employeeKey,
   units)` verifies the participant relationship (a live Employment at
   the business whose Position basis is `per-settlement`) and pays
   `units × rate` as a **`wage`/`piecework`** posting from the Business
   account (payer-derived payability: an NPC's account opens at the
   employer's `banksAt`; an unbanked player is refused — see below). No shipped venue consumes
   it yet — the mine is the named future consumer; it is exercised
   against a test Business (the systems-over-content stance: no fake
   venue authored to demo it).
3. **share-of-flow** (commission/royalty — the consignment/corpo-royalty
   split, now nameable on an employment arrangement):
   `EmploymentApi.flowSplitsFor(business, amount)` yields one
   remittance split per live share-of-flow Employment
   (`floor(share × amount)`, category **`commission`**, Σ capped below
   the flow), wired live at the one revenue seam —
   `OrderController` appends them to the drink `Charge` before
   `BankingApi.settle`. Empty for all shipped content (no authored
   Position carries the basis) — byte-identical today, the same trick
   the consignment split proved.
4. **residual** — **not a clause at all**: ownership, the P&L
   remainder — the proprietor's **draw** (`BankingApi.payDraw`, the
   wallet-afforded `draw <amount>` verb; the business resolves *from*
   the acting proprietor via `businessOfProprietor`, nothing to spoof).
   A distinct, **solvency-checked** `draw` leg kind — never silently a
   wage; see [banking.md](./banking.md) § The leg-kind vocabulary.

**Custody is authored + derived, never defaulted.** A Business carries
`banksAt` (where it banks — a term of its arrangement; missing =
refused), and every account touchpoint routes through
`EmploymentApi.operatingAccountOf(business)`. Worker payability is
**payer-derived** (`ensurePayableWorker`): an NPC with no account gets
one opened at the employer's bank ("your first account opens where
your first money comes from"); a **player is never silently signed
up** — an unbanked player's shift wage is skipped with a warning, a
piecework settle refuses, a flow-split doesn't fire, until they open
their own account at a branch. See [banking.md](./banking.md) § Every
account names a real custodian.

The gig half of the work system (clauses, escrow, the job board, the
two-beat turn-in) lives in [contract.md](./contract.md).

## Deferred seams (named, not placeholders)

- **Player tending** — ✅ **CLOSED** by the trades-and-labor build. It was
  blocked by build-time `MakerMixin` composition on `Avatar`; retiring the
  marker removed the block entirely, because `isFulfilling` composes
  nothing and an Avatar answers it identically to an NPC. A player on shift
  in a `fulfills` seat IS the resolved maker. ⚠ What `order` then asks of a
  *player* maker — craft it yourself, or the engine crafts as you — is
  crafting.md's question, not this one's.
- **Hire/fire drivers** beyond Dave's cover (a fuller management brain) — the
  `hire`/`fire` Api exists; v1 driver is seed authoring + `covers`.
- **The `patronize`/recirculation loop** — off-shift staff at the rail
  ordering/paying/tipping; restores the three-way presence (`railStool`).
- **Shift-change ritual** (count-out / reconcile / handoff), **per-drink tip
  attribution** (route to the `CraftedMixin.maker` of the last drink),
  **multi-tender coverage** (`required-on-shift > 1`), and NPC
  **auto-collect** at shift-end.

## ⭐ Who answers for you — `institutionPath` (2026-09)

`EmployedMixin` carries one thing that is not employment: the **standing
party that fields an actor**, which the harm ledger stamps onto every row
as `killerFor` / `victimFor` ([accountability.md](./accountability.md)).

```
institutionPath() =  an authored `institution:`  →  the employer  →  null
```

⚠⚠ **It is not `directedBy`** — that is *episodic* (a captain's recorded
order began this act) where this is *standing* (you are fielded by X,
order or no order). A guard acting for the watch was not *directed* by
the watch on this occasion, and conflating them would make every
institutional act read as a command.

⚠ **Resolved from the DECLARED affiliation, never the current
location.** A guard who walks into a tavern does not become the tavern's.

⚠ **Synchronous, and that is load-bearing** — combat appends its
accountability rows in the *synchronous* prefix of the beat, because the
coup choreography reads the ledger in the same turn as the killing blow.
That is also why the design's third tier (`ParcelApi.ownerOf(<declared
home>)`) is deferred: it is async, it has no consumer today
(`_domicileAddress` is authored on exactly one row in the shipped world,
and she is employed), and two of the three owner kinds would be wrong
anyway — a wizard `group` and a `player` are not institutions.

⭐ **Tier 2 was free.** The identity build expected to have to write the
person→organization reverse lookup (`organization → people` shipped;
`person → organization` did not). `getActiveEmployment()` was already it.

⚠ It shipped as its own `AffiliatedMixin` and was **folded in during
review**: a mixin whose composers are exactly one class (`Character`) is
the mixin-on-the-wrong-host tell, and both shipped tiers are
authored-or-employment. Split it back out if a third tier over ground
title lands, or if a non-employable host ever needs fielding — by then
there is a second composer to justify it.

⚠⚠ **Only a BUSINESS roster materializes an `Employment` record.** The
roster tick enumerates `mixin.BusinessMixin` and nothing else, so a plain
`Organization`'s `rosterSlots:` resolves to nothing at runtime — which is
why a role fielded by a body that does not trade (a watch) states its
`institution:` outright, and why `lint:identity` credits only the former.

## A wage is a loan or a refusal; vacancy and closure (economic bootstrap, 2026-09)

> Full treatment: [credit.md](./credit.md).

- **`EmploymentApi.payHouseWage(business, workerKey, amount, category?,
  memo?)`** is the one seam a house pays a wage through (the roster tick,
  `house payroll`): arrears first; a rung-2 working-capital draw where the
  house's ledger has earned one; else the wage is REFUSED, recorded on
  `Business.payrollArrears` with the worker as creditor by name, and the
  proprietor told. A landed wage discharges the worker's Arrival Note.
  `Business.charter: [bank]` marks the one chartered lender.
- **The opening advance**: `operatingAccountOf` asks
  `ContractApi.openingAdvance(businessPath)` on a history-less account — a
  0% loan of `treasury.openingAdvance` (a FLOAT, fifty) from the treasury.
- **`Business.getAccountPath()` and `Organization.getOrganizationPath()`
  read the IDENTITY path** (a content row's template path; a minted
  business's own — the rented stall), and the employment-record write
  contract keys on it too.
- **`vacated`** joins the statuses (`EXITED_STATUSES`): a member holder
  away past `employment.absenceVacatesAfterDays` is vacated by
  `EmploymentApi.vacate(actor)` — run by `bringCurrent(business)` (the
  roster tick for every house; `ensureOperatorAt` and `house roster` for
  one) for a lingering holder, and by the estate touch at a return. A
  terminal exit the roster never resurrects; the house card leaves the
  wallet.
- **The closed sign** — `Business.isClosed()` / `setClosed()`, written by
  `bringCurrent`: every principal who could run the house (the entity
  member, every member holder) away past `estate.dormantAfterDays`, and no
  NPC on a position. The counter's `requestAttention` answers `closed`,
  `tickBusiness` runs no shifts, the account refuses outflows. An NPC-run
  house never closes.
- **`house`** grew `book` (loans, terms payable, arrears), `price <thing>
  <ask>`, `roster` (the chart: holders, on/off shift, `vacant`, the closed
  sign), and `stockSheetFor` unions the operated counters' supplied lines.
  `resolveHouse` matches a business operating a FIXTURE in the room (the
  bank's counter in the hall), not only the room.

## History

Built phase-by-phase (Jul 2026, `3785a763..4f24c15a`) from a since-retired
plan. Notable design→implementation shifts:

- `MixinApi.isMaker` was **not** active-aware (the plan's one load-bearing
  assumption); the fix routed it through `isActive` (flipping both consumers
  at once) rather than the surgical fallback. ⚠ Historical: `isMaker` and
  `MakerMixin` were retired by the trades-and-labor build — see *Capability
  grant* above.
- The `EmployedMixin` mutators were first gated `ApiOnly`; the antipattern
  sweep (Jul 2026) re-gated them to the participant contract above and
  moved the transitions onto `BusinessMixin`.
- `OrderController` income re-keyed to the Business account too (the plan
  only named `HouseController`) — required for a combined P&L.
- The Business stands up **lazily** (derived from `operatingLocations` via
  `ensureOperatorAt`) rather than a bootstrap manifest entry or a
  `postRegister` clone, and fixture resolution moved to MQL `peers` + type
  filter (both from MR review; the lazy standup finalized in the Terminus
  build).

> **Work-contracts build (`3969a34e..d8389518`).** Added the
> compensation-bases section above (the `compensation` term on
> `Position`, `settlePiecework`/`flowSplitsFor`, the draw), the authored
> **`banksAt`** custody term + `operatingAccountOf` seam, and the
> payer-derived `ensurePayableWorker` rule (NPCs open at the employer's
> bank; players are never silently signed up). The gig half lives in
> [contract.md](./contract.md).

> **Organizations build.** Factored the org chart out of `Business` into
> `OrganizationMixin`, generalized `proprietorPath` into the polymorphic
> **appointing authority**, added nesting and the **`appoint`** verb, and
> renamed `Employment.businessPath` → `organizationPath` (legacy key still
> read on hydration; no migration). The Business half — `banksAt`,
> `operatingLocations`, the P&L account, wage settlement — is unchanged,
> and no existing assertion in the employment, Dave's Bar, civics,
> Terminus or banking suites was edited. Its first consumer is the press
> substrate ([press.md](./press.md)); the obvious next one is `Corpo`,
> which cannot answer *who runs Veshko?* while positions live only on
> Businesses.

## ⭐ The employee roster — a memo, never a warm

*Who works at this organization* used to be answered by reading **every
object in the world** and asking each one whether it held a record here.
It ran on the two hottest money paths — `holdersOf` (every position read)
and `flowSplitsFor` (every revenue moment) — so the cost was the size of
the realm multiplied by how often anybody got paid.

`EmploymentLogic` keeps `organizationPath → the durable identities of
everyone with a record there`, and `EmploymentApi.employeesOf(path)` is
the read. Four properties, each load-bearing:

- **Terminal records stay in.** An exit has to be *visible* for
  `holdersByPosition` to suppress the matching authored roster entry, so
  the contract is *everyone with a record*, not *everyone employed*.
- **Filled lazily, and ALL AT ONCE.** One enumeration fills every
  organization's set and flips a `rostersFilled` flag, so a second
  organization's first read costs nothing — and an organization absent
  from the map afterwards genuinely has nobody, which an empty set alone
  could not distinguish from a miss.
- **Never warmed.** Nothing at boot is responsible for it; a logic
  reload costs one re-derivation. (A roster nothing warms reading empty
  forever is a mistake this codebase has made three times.)
- **Maintained by a witness, and safe when stale.** `EmployedMixin`'s
  `employments` is an **accessor pair**, so every write lands on the
  setter — a hire, an exit, the Hydrator's bracket-assign, the
  persistence spine's restore. Nothing is ever removed: a leftover entry
  resolves to no record and every reader skips it.

⚠ **The setter calls a METHOD (`_noteEmploymentChange`), not the logic
directly.** The witness is gated on *the actor writing its own
relationship*, and an accessor is not a dispatched frame — a bare call
out of the setter is attributed to whoever did the assigning (a test,
the Hydrator) and denied.

Businesses are found the same way: `businessByLocation` and
`businessByProprietor`, filled by the one enumeration that fills the
business cache, with today's rebuild-and-retry on a miss for a business
that post-dates the memo.
