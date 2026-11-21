# Storage — goods held as a count, in somebody's keeping

A **store** is a holder whose contents are a **ledger**. An icehouse does
not hold ten thousand blocks of ice; it holds twenty tonnes of black ice
and two tonnes of dry sawdust, and the world models exactly that — one
**line** per grade, as an amount. Shipped by the storage-and-ice build
(2026-10), with three composers on day one: the ice trade's **stack**, the
haulage **depot** (re-based onto it), and carpentry's **sawdust** as the
packing a stack is kept in.

It is kernel substrate (`lib/storage/`) because its composers sit in three
packs with no common ancestor. The pieces:

| concept | what it is |
|---|---|
| `StoreMixin` | the ledger, the face, the reconcile, the heat sink, custody |
| `StoreLine` | one line: a pool key, an amount, one unit's mass, the pooled state |
| `PoolableMixin` | the GOOD's declaration of how it pools and what time does to it in store |
| `StoreReceipt` / `ReceiptTerms` | a bailee's acknowledgement, on printed terms |
| `StoreBookMixin` | the house's book — the instrument custody is done with |
| `Stockpile` | the kernel's instanceable store: `PersistableMixin(StoreMixin(Holder))` |
| `StoreBook` | the instanceable book; affords the `store` verb |

⭐ **The store is custody; it sells nothing.** A counter (`Stock`, see
[retail.md](./retail.md)) always sells instances; a counter LINE may name a
store as where it is refilled from. Those are not two shop models — they
are a shop and its back room.

## ⭐⭐ A line, and the good that says how it pools

A line is keyed by `poolKey()` alone, and a pool key names the **good by
what it is, not by its row**: ice's is `ice|<quality>` whether the object
is an ice block or an ice lump. A lump of pan ice and a squared block of
pan ice are the same matter at the same grade, so they share a line — the
store holds frozen water by quality, and shape is not a property the melt
or the receipt cares about. Sawdust pools by its wetness band
(`sawdust|dry`, `sawdust|sodden`), so wet packing is its own line.

The line's `goodPath` is the template of the **first** deposit (or the
seed), and is what a withdrawal rematerializes. A house that took in lumps
pays out blocks of the same grade at the line's mass per unit — honest,
because commingling means a depositor cannot demand *their* lumps back.

`PoolableMixin` is the good's declaration, as instance methods the store
calls on a line's face — the `Stackable` `canMergeWith`/`onMerged` shape:

| method | answers | ice | sawdust |
|---|---|---|---|
| `poolKey()` | the good + its band | `ice\|black` | `sawdust\|<wetness band>` |
| `poolState()` / `applyPoolState()` | what the pooled matter keeps | `{temperatureK}` | `{saturation}` |
| `mergePoolState()` | two states of one key | mass-weighted (the base) | mass-weighted |
| `applyPoolKey()` | stamp a line's grade onto a fresh unit | sets the quality | — |
| `poolRole()` | `stock` or `packing` | stock | **packing** |
| `reconcileInStore(ctx)` | what a step of time does to the line | the plateau melt | inert |
| `storeAnchorK(state)` | the line's thermal anchor while it exchanges | 273.15 K at its plateau | `null` |

The base answers are inert — a good that pools by its row, keeps no state
and does nothing in a store — so a composer overrides only what is true of
it. ⚠ `PoolableMixin` is on `IceBlock` and `Sawdust`, **not** on `Good` or
`Casting`: a metal lump pools by nothing here.

## ⭐ The face — a count you can `get`

`get` and `put` bind real objects, and a count cannot be bound. So for
every line with stock the store keeps **one** real exemplar in its own
contents — the top of the stack, a clone of the line's good at one unit's
mass carrying the line's state. **The line's `amountKg` includes the
face.**

- **`put <good> in <stack>`** (`in`, never `on` — a stockpile is an open
  `Holder` with no `on` region) moves the good in; the store **absorbs** it
  into its line once the move settles (or it becomes the face of a new
  line). Arrivals settle in a scheduled step *after* the move completes,
  because the stack merge-on-arrival would otherwise fold a sawdust heap
  into the face before the store saw it.
- **`get <good>`** takes the face out: the line falls by the face's mass,
  **no receipt changes** (off the books), and the store re-faces.
- A face that was **destructed** (an ice face that melted away between
  reads) is not debited again — the line's own physics already took it. A
  face is debited only when it left *to a container*.
- Every reconcile **re-stamps** each face from its line. The face is a
  VIEW of the line; whatever its own object clock did meanwhile is
  discarded.
- A restore re-adds the faces as contents. The store skips arrival
  accounting inside the persistence spine's restore frame and adopts faces
  by key — otherwise every restart added a unit.

**Who put it.** `ContainmentApi.move` carries no actor, so the putter is
read in the store's own veto (`canAddContainable`) while the item is still
in their hands, and consumed when the arrival settles. A put by **staff on
shift** for the house raises the **house's claim**; anyone else's put
raises the stock and no claim — pilferage in reverse, which is also
honest.

`look <stack>` describes the ledger in **words** — fullness against
capacity in bands, the packing's wetness — never a number.

## What a store accepts, and how much

The container's own veto, asked by `put` first and asserted by
`ContainmentApi.move` (which is what stops a brain's forced put overfilling
it), refuses in the store's words:

- **not poolable** — *"That does not go in a stack like this."* — unless
  the store `acceptsLiveGoods` (the depot), which holds a unique good
  **live**, keyed `live:<chattelId>`, one receipt each;
- **not what the building is for** — `acceptsPoolKeys`, each entry a pool
  **kind** (`ice` takes every grade) or one **key** (`ice|black`); `null`
  takes any pooled good. The icehouse stack authors `[ice, sawdust]`, so a
  sack of grain is refused at the door;
- **full** — *"The house is full."* Capacity is **mass**, every line
  counted, packing included (`capacityKg`, 0 = unbounded).

A full store is a legitimate world state, refused at the verb.

## The lazy reconcile, and the heat sink

Every read of a store (`look`, the stocktake, every custody act, a
counter's draw, the veto) brings each line current from its
`lastReconciledS` to now — nothing ticks. The step:

1. **The outside air over the gap** comes from the weather walk, segment
   by segment (the walk the ice on a still reach uses), never from the
   room's stale outside temperature, which is restamped only where
   somebody is.
2. **The room's temperature** per segment is the room's own answer —
   `envelopeSteadyStateK(outside, at)`, the closed-form steady state of the
   room's envelope, which already includes this store's pull on the air.
3. **Per stock line**, an exposed surface
   `stackShapeFactor × (amountKg/ρ)^⅔` (default 5: a stack on a floor shows
   four sides and a top), buried in the packing at a depth of the packing's
   bulk volume over that surface, conducting at the packing material's
   **wet** conductivity (`Material.wetProperty`, the one wet law). The good
   answers `reconcileInStore` with its new mass, state and meltwater.
4. **Meltwater soaks the packing**: what does not drain (`drainage`,
   default 0.5) raises its saturation by the water over what the packing
   can hold. A soaked packing conducts like the water in it — **the spiral
   a keeper repacks to stop.**
5. `lossKgPerDay` is updated from the step; a line that has emptied is
   dropped and its face destructed; the store captures itself.

⭐ **The heat sink is not a special case.** A store is a
`RoomHeatExchanging` occupant: a conductance from the room's air, through
its packing, to the anchor of every line at its plateau (melting ice:
273.15 K). The envelope sums every occupant's exchange — the space heater
pushes, the store pulls — so an icehouse reads colder than the same shed
empty, with nothing anywhere naming ice. The store never names ice either:
the anchor is the good's `storeAnchorK`. The physics of the exchange,
the envelope and the plateau are [thermal.md](./thermal.md)'s.

The arithmetic the unit tests pin: 20 t of ice in 2 t of dry sawdust in a
48 m³ shed at 20 °C outside loses ~24 kg/day, the house settling near
10 °C; thin packing loses more; soaked packing more again; a 2 t house
loses a share several times larger. Bigger houses keep better —
derivable, not authored.

⚠ **Only the first packing line insulates.** Dry sawdust put beside a
sodden line changes nothing until the sodden line is gone, and the
keeper's brain digs out a kilogram a beat. A real repack wants the packing
lines combined (depth summed, conductivity mass-weighted) and a bulk
dig-out — a mechanism gap the build's drive recorded, owed to
storage-slate.

## ⭐⭐ Custody — receipts on the book

A **receipt** is the bailee's acknowledgement that it holds goods for a
depositor: `{bailee, store, depositor, good, poolKey, unit, amountKg,
originalAmountKg, unitMassKg, terms, what, filedAtS}`, plus
`settledAtS`/`shortfallPct`/`withinAllowance` once it is paid out. It is
filed as the existing `warehouse-receipt` business document under the
bailee's own branch (`<business>/warehouse-receipts/<id>`) — no new
collection, and the closed business-paper kinds stay at three. A depositor
keeps one running receipt per line; a second deposit of the same grade
adds to it. A unique good is `unit: count`, for one.

The **terms** are printed on the store row and copied onto every receipt:
`allowancePct` (the shrinkage that counts as ordinary loss; default 10) and
`feeMinorPerKg` (what taking it back out costs; the icehouse charges 1).

⭐ **Pooled goods are owned in common.** A receipt is a claim on a line, not
on particular blocks. **The payout rule**: for a line holding `S` with
claims `C` on it (the house's included), `f = min(1, S/C)`; a withdrawal
pays `min(want, claim × f)` in **whole units**, and the claim is debited by
what was CLAIMED (`paid / f`). So a run on a short store pays everyone pro
rata until settled. **The allowance decides nothing about the payout** — it
classifies the shortfall on the record: settlement writes `shortfallPct`
and whether it was `withinAllowance`, and the withdrawal says so when it
was not. *The engine reports; the courts judge.*

### ⭐ The house's claim

The business that keeps the store is the **bailee** (`businessPath`), and
it is also a depositor: **the house's claim** is a receipt in its own name,
one running receipt per line. It rises when the row's seed is laid down
(a store's seed is the house's own stock), when staff on shift `put` stock
on the stack, when staff on shift `store deposit`, and when the house buys
a good through a store-fed counter line (`sell`). It falls when staff
withdraw and when a counter brings stock up — **`drawForHouse`**, which
mints units off the line on the house's claim, debits that claim, and
notes the draw in the line's `houseDrawKgPerDay`, apart from depositors
taking their own goods home. An empty claim draws nothing: *an empty store
means an empty counter.*

⭐ The house is a depositor like any other: `drawForHouse` pays its claim
at the same pro-rata fraction `withdraw` pays (`min(1, S/C)`), so on a
short line the counter's bring-up never draws ahead of the depositors —
no run. (It drew at face value until the pre-merge audit.)

### The keeper, and the seat that gates the act

Somebody has to be keeping the book. A custody act asks, in order: is the
store kept by a business at all (*"Nobody keeps this store."*); is the
actor **staff on shift** for it (`Employed.isOnShiftFor(business)`) — who
needs no keeper and acts on the house's claim; else is there a
**custodian** in the actor's room — an occupant employed by the house, on
shift, in one of the row's `keeperPositions` (any position when none are
authored). None: *"Nobody is keeping the book just now."* — refused in
words, never a silent no-op.

Before asking, the store **brings the house current**
(`EmploymentApi.ensureOperatorAt`) — the counter's own rule before a sale,
because a shift flips on the roster pass and after a quiet stretch the
keeper read as off at noon.

## The verbs

Custody is **one noun with subcommands**, as `bank` is: a deposit, a
withdrawal and your receipts are three entries in the same book, kept by
the same keeper, under the same printed terms. ⭐ The rule adopted in
review: *an act on the world is a bare verb; a transaction with an
institution's book is a subcommand of the institution's noun.* (The
verb-space refactor this implies is `command-taxonomy-slate`.)

| verb | what it does |
|---|---|
| `store` / `store receipts` | what the book holds in **your** name here |
| `store deposit <thing>` | the keeper takes it in and writes you a receipt, printing the amount and the terms |
| `store withdraw [<n>] <good>` | the keeper pays out against your receipt in whole units, charges the fee (never to staff; the fee reported is the fee taken), and says if it settled short |
| `put <good> in <stack>` / `get <good>` | the stack as a holder — off the books (the house's claim rises only for staff on shift) |
| `look <stack>` / `look book` | the ledger in words; the outstanding claims by line |
| `measure stock` / `analyze stock` | the **stocktake** (below) |
| `buy` / `sell` at a store-fed counter | retail's acts — see [retail.md](./retail.md) |

The verb is afforded by the **instrument that performs the act** — the
house's `StoreBook`, standing beside the counter, which confers `store` on
whoever shares its room or holds it — never by the keeper and never by the
room. The book names its store (`storePath`). `store withdraw`'s `<what>`
is one greedy string (`3 ice`) the controller splits; a withdrawal matches
a receipt by the good's row leaf or any word of it, the pool kind, the
pool key, or a word of what was deposited. The controller is dumb: the
store owns the whole sequence and returns its refusals.

**The stocktake** is the kernel reading `stock` (see
[instrumentation.md](./instrumentation.md)): `measure stock <stack>` gauges
each line with a gauging rod, bracketed to the reader's **appraisal**;
`analyze stock` is the eye, in words. A block taken off the stack by hand
changes no receipt; only a stocktake shows the gap. The keeper compares
the reading with the book by eye — **the engine prints no verdict.**

## The economy

A store-fed counter line (`from: { store: <stack> }`) is never cloned: the
reset sweep and a `buy` on an empty shelf **bring it up** to order on the
house's claim. Its price is one formula, `base × (1 + e × scarcity)`, with
scarcity asked of the source: for a store-fed line, the house's **days of
supply** — `Store.houseDaysOfSupply`, the house's claim paid pro rata,
losing its share of the melt and drawn at the house's own rate — against
`retail.stockingHorizonDays` (120). A house that is running down asks
more; one with years of supply asks base. ⚠ With no claims read yet (a
store not yet reconciled) the whole line counts as the house's, as a seed
leaves it.

The house also **buys** (`sell`, `buys: true` on the line): its bid is
derived, never authored — the ask ÷ (1 + `retail.termsMargin`) — so the
house pays more when it is short and less as its stock rises. ⭐ The market
is self-limiting with no cap anybody authored: the bid falls with supply,
the store's capacity refuses past full, and the house's money runs out
(the counter says so on `look`, before anybody makes the trip). Imports
are the backstop — a good nobody stores is cloned at the sweep. Details:
[retail.md § The counter](./retail.md) and § The house buys.

## Persistence

`Stockpile` is the `MarketStalls` shape: one fixture per row, its own
`holder_snapshots` record restored at hydration. The ledger (`lines`) is
the record and the faces ride with it as contents; the receipts are
documents. ⭐ The row's **`initialLines`** is a **`seed: true`** field —
the template applier's mint-only phase — and is laid down by the hydration
source only when no record restored a ledger. (It shipped first as an
`authorable`-only field, which the applier's phase 1 silently drops: both
Hollow Mere houses booted empty and four drive checkpoints failed on it.)
A seed line names `goodPath`, `amountKg`, and optionally a `grade` and a
`state`.

## Content

- `trade-ice`: the **stack** kind (`/trade/ice/thing/stack` —
  `acceptsPoolKeys: [ice, sawdust]`, `keeperPositions: [keeper]`, terms
  10% / 1 a kg) and the **book** kind. Hollow Mere's two houses extend the
  stack — the large house 30 t seeded with 20 t of black ice in 2 t of dry
  sawdust, the small house 3 t with 2 t in 0.2 t — and the icehouse counter
  sells ice brought up from the large stack and buys it back.
- `trade-haulage`: the depot's `Warehouse` is `StoreMixin(Holder)` with
  `acceptsLiveGoods` — the first bailee store, holding crates live; its
  programmatic intake below the custody act is `depositFor`, and the
  depot book affords `store` at the counter.

A second icehouse is rows: a stack row extending the kind, a book naming
it, a keeper seat.

## What it does NOT do

- **Sell.** A store is custody; the counter sells.
- **Judge a shortfall.** It reports pro-rata payment and the allowance on
  the record; nobody is blamed by the engine.
- **Tick.** Nothing runs while nobody reads; the reconcile is lazy and
  exact over the weather segments.
- **Name ice.** Every ice fact (the plateau, the anchor, the grade) is the
  good's answer to `Poolable`.
- **Hold a store in a place.** A store is an object in an authored room,
  not a location.
- **Mint a collection.** Lines ride the holder snapshot; receipts are
  business documents.

## Deferred seams

- **Store-to-store transfer, the ship's hold, cargo; handling at scale**
  (poling, hooking, the elevator; tongs as a gate) → `freight-slate`.
  `drawForHouse` and `deposit` are the two ends; a transfer is a draw at
  one and a deposit at the other with a bill of lading between.
- **Damp, spontaneous combustion, vermin** → `storage-slate`. Attach:
  `reconcileInStore` receives `StoreConditions`; a humidity term, a heat
  source on the good, and a `pests` term (cleanliness's floor pests) are
  the seam; `poolRole` admits a third role. Spoiled packing a keeper digs
  out goes to the midden.
- **The repack** (packing lines combined, a bulk dig-out) → `storage-slate`.
- **Transferable receipts** (futures, collateral) → `storage-slate`;
  `StoreReceipt.depositor` is the field a transfer re-writes.
- **The bank vault as a store** → the banking / `currency-slate` (recorded
  in storage-slate § 4).
- **The timber yard, and seasoning as pool state** → `storage-slate`
  (candidate): `PoolableMixin` on the board with `{seasonedFraction}` and
  the minimum merge.
- **Natural-ice contamination** (sewage-fouled river ice) →
  `cold-chain-slate`: a cut block carrying the reach's contamination, and
  the line's state carrying a load the way sawdust carries saturation.
- **NPC restocking unobserved** → `unobserved-economy-slate`; the physics
  needs nothing.
- **The building's upkeep** → the repair build (materials weather and
  `maintain` relays them; the icehouse opts in once a supply reaches it).

## History

Built by the storage-and-ice build, MR !378 (2026-10-17/18): W2 the store
substrate, the face and the heat sink (`6faded846`), W3 custody and the
stocktake with the depot re-based (`a27431ee3`), W4 retail from a store
(`3534ec972`). In review (2026-10-18) custody was folded from two bare
verbs, `deposit` and `withdraw`, under the one noun `store` with a new
`store receipts` (`26987a135`, `e1d0462cb`); the counter's line took one
source (`from: { supplier } | { store }`), the house's own days of supply
(`houseDaysOfSupply`, its draw kept apart), a kind-or-key
`acceptsPoolKeys`, and the house learned to buy (`sell`).
