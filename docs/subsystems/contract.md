# Work contracts — the gig substrate

> **Status:** built on `feat/work-contracts`. The buildable kernel of the
> livelihood slate's work model: the clause primitive with engine
> verification, the gig lifecycle over conserved escrow, the physical job
> board, and the two-beat turn-in. The arrangement generalization of the
> employment engine (compensation bases, the draw) is documented in
> [employment.md](./employment.md) § Compensation bases; the banking legs
> in [banking.md](./banking.md) § The leg-kind vocabulary.

The income backbone the platform doctrine requires: kill→reward is
severed, livelihood comes from **work** — posted, claimed, verified,
paid. Gig and employment are two shapes of one work system, so the
substrate lives in the one common `lib/employment/` namespace (the
`lib/standing/` precedent: one consolidated lib home read by several Api
pairs), with its own gated face — `ContractApi` (`api/contract.ts`) →
`ContractLogic` (`/obj/api/contract`) beside the shipped
`EmploymentApi`/`EmploymentLogic`.

## The clause primitive

A **clause** is the first-class unit of work: `{shape, condition}`
(`lib/employment/Clause.ts`), where the condition is a query over
**modeled world-state, evaluated by the engine** — never an agent's
diegetic witness. Two shapes:

- **achieve** — make the condition true once; structurally settleable →
  escrowable, the gig's shape. A v1 Contract holds exactly one.
- **maintain** — keep it true over an interval; structurally
  *unsettleable* → paid by interval. **The shipped time-wage IS the
  maintain shape's paid-by-interval instance** (a bartender maintains
  "the bar is tended" across a shift). No violation-detection engine
  ships until a consumer (a bouncer-shaped job) exists; when it does,
  the mechanism is a direct witness hook on the relevant chokepoint (the
  `_registerMergeOnArrivalHook` pattern), never `EventApi` for a
  single-consumer seam.

**The contract boundary**: a clause may only be escrowed if its
condition is engine-verifiable — `Condition.validate` admits only the
**closed template vocabulary** (`CONDITION_TEMPLATES`, v1:
`delivery`). Free-form intents (and free-form MQL — the anti-grief
boundary) are rejected from the system-backed path. The vocabulary is
the extension seam later builds widen (`cull`, `escort`, `restock` —
each one new `holdsFor` predicate, never a new engine seam).

**Delivery** (`lib/employment/Condition.ts`): *item X rests in/on
destination Y*. The item ref is instance-bound (`chattel`, the durable
`_chattelId` — deliver *this* crate) or kind-bound (`template` — any
clone); a `Globbable` is refused outright (a merging stack has no stable
identity — the chattel precedent). `holdsFor` is the authoritative
check: a bounded **upward** containment walk comparing ancestor
`templatePath`s to the destination (a crate in a chest in Dave's Bar
delivers), **refusing any Creature ancestor** — *strict possession*:
still-carried is not delivered — plus the `restingOn` surface leg
(`placeOn` puts an item in the *room* with a `restingOn` pointer;
without this leg "deliver to the counter" could never hold).

## The lifecycle + storage

States: `open → claimed → settled | breached | expired` (three
terminals). A failed **claim** (abandon / claim expiry) breaches the
claimant and **reopens** the gig — it does not terminate it.

Storage is the chattel/parcel precedent: a `contracts` current-state row
(`ContractRecord`) + an append-only `contract_events` chain
(`ContractEvent`: `posted`/`claimed`/`fulfilled`/`breached`/
`released-back`/`settled`/`reverted`), one writer (`ContractLogic`),
parties keyed on durable `templatePath`s (`ContractParty`, player |
business — the `ChattelOwner` shape widened). **Money legs live only in
`bank_ledger`**; events carry the linking `txId` (`fulfilled` carries
none — it is the engine-sealed proof-of-delivery). Unlike
chattel/parcel there is **no registry Stuff**: those registries exist
because title is consulted synchronously by `AccessApi`; contract state
has one writer and all reads are async record finders — with turn-in
verification there is no watch index and no warm cache at all, and
`ContractApi` has **no `boot()`** (nothing warms, nothing subscribes).

## Escrow — conserved banking, not a new store

The reward is a **real held balance** moved only by the sealed
`postTransaction` chokepoint: `escrow-hold` at claim (exclusive) or post
(open-bounty), `escrow-release` to the completer at settle,
`escrow-revert` to the issuer on breach/expiry — held in the
**per-contract real account** `escrow:contract:<id>` (owner = the
contract, **custodied at the issuer's own bank** — wherever the funding
account lives, *your* bank holds *your* stake; the future
escrow-as-a-bank-product seam on Terms). Why not a sentinel, not a pooled account, and not a per-contract
Stuff — and why `reconcile` stays green with escrow in flight with zero
audit changes — is argued in [banking.md](./banking.md) § The leg-kind
vocabulary. Posting fails if the issuer can't fund it; **no credit,
anywhere**. `escrowClose` deletes the zero-balance row at every
terminal, so live escrow rows scale with open contracts.

Issuer resolution (actor from context, never a parameter): a player
posts from their **primary** account; `--business` resolves
`EmploymentApi.businessOfProprietor(actor)` (an NPC proprietor
dispatches the same verb) and escrows from the Business account.

## Claim modes

- **Exclusive** (the delivery default): `claim` escrows and locks the
  gig to the claimant, with a **game-time claim expiry**
  (`contract.claimExpiryGameHours`) — squatting cannot block a board.
  Only the claimant may `fulfill`/`complete`. If the hold fails at claim
  (funds moved since posting), the gig closes `expired` with a
  `reverted`/`unfundable` event — the board never advertises a check the
  system can't cash.
- **Open-bounty**: escrowed at post, no claim step; anyone may
  `fulfill`/`complete` — attribution is inherent (the actor is the
  command giver), a `fulfilled` row redeems only for its own actor,
  first verified settle wins, a concurrent second `complete` is refused
  by the compare-and-set terminal-state guard.

Both modes take an optional posting expiry (escrow reverts to issuer).

## Verification — the two-beat turn-in

Completion is an **explicit player petition**; the engine checks
`holdsFor` against modeled state **at that moment**, viewer-blind — the
engine-as-observer doctrine. **No ambient detection**: no containment
hook, no `EventApi`, no watch index, no boot subscription — the
containment chokepoint is untouched, and a new job template is a new
predicate, never a new chokepoint instrumentation. (A pinned test moves
the crate into the destination and asserts *nothing settles*.)

1. **`fulfill <id>` — the capture (proof-of-delivery).** At the
   destination (the presenter's environment chain must include it — the
   handoff is diegetic, not remote). Runs `holdsFor` now; on success
   appends the engine-sealed **`fulfilled`** row (actor, no money, no
   state change). Trustworthy by construction: only gated logic writes
   it, only after the engine itself verified — call security is the
   authentication. The narrowest instance of the future general
   **trusted-recording instrument** (the "tricorder"; the courts'
   evidence substrate).
2. **`job complete [id]` — the redeem (payday).** At the board; settles
   on **either** live `holdsFor` **or** a valid post-claim `fulfilled`
   row whose actor is the completer — the payout survives later state
   drift (the recipient taking the crate inside is the *point* of
   delivering it). On verification: the payee resolves **before** the
   terminal flip (a settled-but-unpaid record is unreachable) — a
   **player must already hold a primary account** (never silently
   signed up for a bank; the refusal is the nudge to open one — *gig
   settlement in coin* is a named deferred seam), while an NPC payee
   gets an account opened at the bank custodying the escrow
   (payer-derived) — then the CAS state flip, escrow release, `settled`
   recording attribution both ways, `escrowClose`.

## Expiry is lazy (observe-first)

Every touchpoint (`openGigsOn`, `claim`, `abandon`, `fulfill`,
`complete`, `contractById`) first runs `expireStale`: a claim past
`claimExpiresAt` → breach + reopen; an open posting past
`postingExpiresAt` → revert + `expired`. No sweep, no counter, no
scheduler (the withdrawal-quota / residency precedent) — conservation
makes laziness safe: the held stake sits in a real account `reconcile`
counts while nobody looks; the refund lands on the next touch. A
game-time safety-net sweep is a named deferred seam.

## Breach is felt, cheaply

Breach (abandon / claim expiry) = `escrow-revert` + a durable
`breached` row naming the claimant + an **issuer-side regard nudge**
(`RegardApi.adjustRegard(issuer's person, contractor,
−contract.breachRegardPenalty)` — the issuer itself for a player, the
proprietor for a business). Best-effort: either side not live (lazy NPC
standup) → the nudge no-ops gracefully; the durable event row is what
reputation consumers read later. No global reputation write.

## The board + the `work` verb surface

`JobBoard` (`lib/employment/JobBoard.ts` — the `lib/retail/Stock`
placement: multiple boards in multiple localities) is the physical
discovery surface. A gig is posted *to a board* (its `templatePath` is
the pool's visibility key — the board is the surface, not the pool).
Content affords content: the board's own `commandContributions` afford
the **`job`/`jobs`** verb (`environment`), first placed in the
**Terminus terminal hall** (a public noticeboard on the station
concourse — where newcomers arrive needing a living;
`seeds/domain/terminus/terminal/job-board.yaml` + the hall's
`populates`; keyworded `noticeboard` — the hall's `board` detail is the
split-flap departures board).

Two verbs in the new **`work`** category (the house subcommand
discipline — one dispatch verb per feature, never a verb per
operation):

- **`job`** — bare = browse (short-id rows, escrow legibility);
  `post <item> to <destination> for <reward>` (`--bounty` /
  `--business` / `--expires <h>`); `claim <id>`; `complete [id]`;
  `abandon [id]`. Ids accept a unique prefix; a bare
  `complete`/`abandon` resolves the giver's single active claim here.
- **`fulfill`** (alias `handoff`) — deliberately a **standalone verb**:
  affordance is per-verb, and the capture beat must travel with the
  courier away from any board — afforded from `self` by the born-with
  `CredentialWalletUpdate` (diegetically, the implant logs the
  delivery). A bare `fulfill` resolves via `ContractApi.activeClaims`.

Controllers are thin (model → the `ContractApi` call; Scene + note);
every actor is context-derived inside the gated logic.

## Dials

`contract.claimExpiryGameHours` (48) · `contract.postingExpiryDefaultGameHours`
(0 = never) · `contract.breachRegardPenalty` (15) — plus banking's
`banking.defaultCustodianBank`.

## Deferred seams (named, not built)

The systemic job generator (the world posts its own needs); the
public-works floor/match (Circulation Reserve dependency); more
condition templates (cull / escort / restock); ambient/event-driven
verification for maintain-violation (a bouncer-shaped consumer first —
direct witness hook, never `EventApi`); standing/competence claim
gates; demand/difficulty pricing; entity forms + differential tax rates
(they read the leg-kind vocabulary); liability scope-context on
`accountability_events`; the client board card; the game-time expiry
safety-net sweep; NPC claiming brains; the mine as the first real
piecework venue; the general trusted-recording instrument (the
`fulfilled` row is its narrowest shipped instance); **gig settlement in
coin** (an unbanked completer paid in physical cash over a counter —
needs an escrow→till bridge at a branch; today the refusal nudges the
player to open an account).
