# Retail — the general store

The first **retailer** business (retail slate S1), and the proving
consumer of the [chattel possession core](./chattel.md). A shop where you
`buy` goods off a bounded, self-restocking shelf (and **own** them, not
just carry them) and `consign` your own goods for another player to buy.
It rides *real* ownership; its consignment loop (custody-vs-ownership) is
what forces the ownership design honest.

Heavy reuse, few new primitives: `Business`/employment
([employment.md](./employment.md)), Attendant + its lease
([attendant.md](./attendant.md)), banking `settle`/`Charge.splits`/tax
([banking.md](./banking.md)), containment, the reset sweep
([residency.md](./residency.md)), and the bar's offer pattern
(generalized). The net-new pieces are the **retail counter**, the
**consignment listing**, and the **shared price-list**.

## ⭐⭐ The priced SERVICE — `Tariff`

A `Menu` prices **recipes** (make me this) and the `Stock` counter prices
**items** (sell me that). Neither can price *something done to what you
already have*, and that is what a repair shop, a clinic and a necropolis
sell.

⚠ **The gap was total.** No shipped priced key resolved to anything but a
recipe or a stock line, so paying for a repair, a treatment or a burial
had **no path at all** — and the one paid service in the whole tree, the
TPA fare, was pack code rather than content. The wreckage a fight leaves
could not become anybody's work.

`/platform/thing/Tariff` = `PricedOfferMixin(DetailedMixin(Thing))` plus
`services: Record<offerKey, ServiceKind>` over a **closed kernel
vocabulary** — `repair` · `treatment` · `burial`. `OrderController`
orchestrates each. Closed and kernel-owned on purpose: a venue that could
define its own service kinds would need pack code, and then a second
venue would need pack code too. ⭐ **A second clinic is a `Tariff` row and
a `Business` row** — the shipped proof is a repair tariff at the
hearthworks smithy and a second one, at a different price, in the
Rejection provisioning shed.

`PricedOfferMixin.collect(key, reason)` is the one settlement path, lifted
out of `OrderController.charge` so a `Menu` and a `Tariff` share it.
Income keys on the **Business** account (the same account shift wages come
out of); `ensureOperatorAt` stands the business up lazily on the first
sale; credential first, then cash. ⚠ **Every failure is "on the house",
never a throw** — no operator, no bank, no funds: the customer is served
and nothing is taken.

> ⭐⭐ **`Tariff.labourIndexed` — care priced by the labour it restores**
> (recovery build, D13). When authored true, a `treatment` service's
> `priceFor` bends by `1 + LABOUR_INDEX × (wage / REFERENCE_WAGE) ×
> shortfall`, read off the ACTING CUSTOMER (both the wage and the harm
> shortfall `1 − Vitals.minCapacityScalar`), so `menu` quotes *your* price
> and the clinic-near-the-mine emerges from who walks in. An unemployed or
> unhurt body pays the base; the humane floor needs no code, because an
> unpaid body still recovers at the free convalescence floor. `labourIndexFor(body)`
> is the unit-tested multiplier; `priceFor` composes it.

### ⚠⚠ The customer is the one who asks

`BankingLogic.settle` derives the payer from **execution context** and
takes no payer parameter. So a service is *bought* (`order treatment`)
rather than billed to a bystander mid-`treat`. That is a real constraint
and the right one: letting one player initiate a debit against another is
a consent question far larger than a priced clinic.

⭐ It is also why **`revive` is not in the vocabulary.** A paid revival
was the obvious third service and it is unbuildable as an `order`:
`requiresEmbodied` names *buy* among the embodied acts a shade loses —
*"Death costs embodied agency and the price of coming back; it never
costs a seat as a person."* A shade cannot purchase anything. The two
routes out — a third-party payer, or an option on `passage` — are both
real design, and neither is decided in passing. → mortal-vessel-slate
(§ *Absorbed from mortality-slate — The re-embodiment service*).

## The shared price-list (`PricedOfferMixin`)

Extracted from the bar's `Menu` so the bar and the store share one offer
model (bar parity a hard constraint — the bar's tests are the gate).
`PricedOfferMixin` (`lib/commerce/PricedOffer.ts`) owns **only** the price
map (`prices: Record<offerKey, minorUnits>` + `priceFor` / `setPrice` /
`pricedKeys`) — Law 1: worth lives on the *offer*, never on the good. What
an offer key *means* stays the consumer's concern (a recipe id for the
bar's `Menu`, an item template path for the store's `Stock`).

## ⭐⭐ The kernel/pack line

> **The kernel provides businesses, corporations, money, an economy. A
> content pack provides a TRADE — and selling over a counter is a trade.**

So the class a row names ships in `trade-shopkeeping`
(`/trade/shopkeeping/thing/Stock`, `…/ConsignmentShelf`), and the
cloakroom in `trade-hospitality` (`/trade/hospitality/thing/CheckRack`).

⚠ **The counter's MECHANISM stays kernel substrate** —
`lib/retail/Stock.ts`, never instanced — because the kernel reads it: the
price index, the credit ladder's rung 0 (`termsPriceFor`), the wage
engine's par read and four controllers all narrow on the base, and a
kernel module may never import a pack. The pack ships the instanceable
twin, which adds only the affordance statics. That is this repo's own
*split it* rule (`CLAUDE.md § Instanceable lives in platform/…`) applied
across the pack boundary.

⭐ **`lint:counters` holds the line at four**, each named in the script
with its reason: `AttendancePoint` (the generic queue — attendance is
substrate; a bank counter, a ticket window and a clinic desk all attend),
`Menu` and `Tariff` (the consumer's `order` surface over a kernel-owned
closed vocabulary), and `BankCounter`, the one to-do — trade-banking is
deferred with a stated reason (credit-slate Parts 6/7). A fifth fails the
build with *a counter is a vocation's instrument; it ships in
`/trade/<x>`*.

⭐ And a store in a **new locality is rows only**: the pack ships a `shop`
archetype (counter · shelf · light · keeper) whose defaults
`Archetype.materialize()` clones, proved by
`trade-shopkeeping/src/__tests__/shop-archetype.test.ts`.

## The counter (`Stock`)

`lib/retail/Stock.ts` (the mechanism; the instanceable twin is
`/trade/shopkeeping/thing/Stock`) — one fixture that is the whole counter: a `Vessel`
(container, holds the shelf goods) that composes `PricedOfferMixin`
(prices), `AttendantMixin` (the storefront lease — instant/scrum for the
newcomer path, a closed counter refuses), and `ResettableMixin` (topped to
par on the reset sweep). The `BankCounter` precedent (a Vessel that
composes its capability).

- **`stockLines`** — authored `{ itemTemplatePath, par, brandKey? }`; the
  offer key is the item template path.
- **`reset()`** (the `ResettableMixin` hook) tops each line back to par by
  cloning fresh goods from `itemTemplatePath` — **items, never money**
  (Law 2). `resetsWhilePresent() → true` (restock while browsed is fine).
- **Self-stocks on standup**: `postRegister` calls `reset()`, so the same
  one mechanism handles boot-stock and ongoing topup.
- **One counter is both** (libations): `Stock` composes
  `ConsignmentShelfMixin` (+ `PersistableMixin`), so a counter with
  **no `stockLines`** (an empty list makes `reset()` a no-op) is a pure
  consignment shelf — the cash-and-carry. `buy` resolves either; a listed
  good on a Stock counter routes to the listing path.

The ownable staples **must be discrete `Thing`s, never `Stackable`** —
chattel stamps a discrete instance, so a fungible stack would fall through
the buy/consign loops (a CI-adjacent content-integrity lint enforces this
on the seeds).

## The buy loop (`buy <thing>`, stamps ownership)

`BuyController` (`platform/idea/cmd/retail/`, the new `retail` command category)
resolves the good off the present `Stock` **or** a consignment listing on
the `ConsignmentShelf`, settles a presented `Charge` (card → cash), hands
the good over, and transfers ownership:

- **stock good** — settle the full price to the store's Business account,
  remit the demo sales tax on it, and **stamp** the buyer
  (`item.stampChattel` — a fresh, author-owned shelf good changes hands).
- **consignment listing** — settle the ask, **split** the remainder to the
  consignor's primary account (the store keeps the commission, its taxable
  revenue), and **transfer** the owner-stamp to the buyer.

Unlike the bar — which floats an unpaid drink — a store hands **nothing**
over unless payment clears. The store account is the **Business** account
(income + wages on one P&L), ensured lazily off `operatingLocations` via
`EmploymentApi.ensureOperatorAt` (the `OrderController.charge` shape).

⭐ **A bare shelf is SOLD OUT, not "isn't for sale here"** (the fishing
build's live browser drive, 2026-09-22). `resolveBuy` only sees what is
physically on the shelf, so a carried-but-empty line used to be refused
as *"float-rod" isn't for sale here* — while the counter's own
`getLong()` still listed it ("On the shelves: float-rod (11), keepnet
(6)"). `Stock.carriesLine(keyword)` answers *does a stock LINE carry
this word*, matched against the template **leaf** — which is exactly
what that description prints — and `BuyController` refuses a carried
line with reason `sold-out` and prose that says so. ⚠ The match is the
leaf only: a player who types a keyword the shelf list does not print
(`cane` for `…/thing/rod`) still falls through to `not-on-shelf`. Every
`par: 1` line hits this the moment somebody buys the last one.

⭐ **`buy` calls `item.followCustody()` after the stamp** (fishing B7,
2026-09) — every other custody verb did; `buy` never had, and a bought rod
vanished at the next restart. Both branches (a stock buy and a consignment
buy). The fish stall (`market/thing/fish-stall.yaml`) is a second
consignment counter with no lines of its own — self-service, capacity 40.

## Consignment — custody-vs-ownership (`consign` / `reclaim`)

The store's brokerage layer over real ownership. With the stamp,
consignment needs no bespoke ownership pointer:

- **`consign <thing> --ask <coin>`** moves **custody** to the shop's
  `ConsignmentShelf` while your **owner-stamp stays put**. A
  `ConsignmentListing` (`{ listingId, itemChattelId, consignorKey,
  askMinor }`) is a brokerage record — ask price + payout target — **not**
  an ownership pointer; "whose is it" is always `item.chattelOwner()`.
  Gates: you must own it (`ownerOf`, not custody); it must be discrete (a
  glob is refused); you must hold a bank account (the payout target —
  nudge to Goodkin otherwise); and you must be under the **per-consignor
  listing cap** (`retail.consignment.listingCap` — the shared-shelf
  anti-grief guard, the withdrawal-quota sibling; the platform ships
  **24**, a case per consigning outfit on a distributor's counter — and
  the `consigns` brain reads its outfit's headroom under it, so an NPC
  hand never runs at an `over-cap` decline). An unstamped
  author-owned good is stamped to you on consign (establish-on-consign).
- **A sale** splits the ask (commission → store, remainder → the
  consignor's **primary bank account** via `Charge.splits`), transfers
  custody **and** the owner-stamp to the buyer, and clears the listing.
  The store **fronts no coin** (a real buyer's coin funds both legs);
  conservation holds (`reconcile().balanced`). The consignor need not be
  online — payout rides `primaryAccountIdOf`, a pure DB read.
  `commission = ask × retail.consignment.commissionRate`.
- **`reclaim <thing>`** returns custody of an unsold listing — **no
  chattel op** (ownership never left you). It authorizes on
  `item.chattelOwner()`, not on possessing any ticket — so a non-owner
  reclaiming someone else's goods is refused (custody without title is
  theft).

### The custody base — `HeldGoodsMixin` (consignment ⊃ coat check)

The custody half of consignment is factored out as **`HeldGoodsMixin`**
(the shared base): move a good into a fixture's custody (owner-stamp
stays put), know whose it is (`recordHolding`/`holdingFor`/`holdingsOf`/
`countHeld`/`removeHolding`), and hand it back to its owner
(`resolveHeld`). *That is the coat check, whole.* `ConsignmentShelfMixin
extends HeldGoodsMixin` and adds the **sale** layer on top — the ask, the
per-consignor cap, `buy` — so a `ConsignmentListing extends HeldGood`
with an `askMinor`. The `ConsignmentShelf` public surface is unchanged;
the sale controllers don't know the base exists.

**The check rack (`check` — the bar-fight build)** composes **only the
base**. `CheckRack` (`trade-hospitality/src/thing/CheckRack.ts` — a
cloakroom is the publican's) = `Persistable +
HeldGoodsMixin + FixtureMixin`, affording `check` + `reclaim`, never
`consign`/`buy` — a checked weapon is a plain held good (no ask, no
listing), so there was never a "not-for-sale" flag to add: a coat check
is the base without the marketplace. `check <weapon>` (gated on
`CombatApi.isWeapon` — a shield is armor, refused) records a holding and
mints a diegetic `Ticket` (the owner-stamp, not the ticket, is the
reclaim authority — a lost ticket never traps your weapon). `buy` finds
no consignment shelf at a rack, so a checked weapon simply isn't
buyable; `reclaim` narrows on `HeldGoodsShelf`, so it serves both a store
shelf and a check rack. The lounge's rack rides `FixtureMixin`'s `seatIn`
self-seat into the Warren host (the TPA-terminal precedent), so it stands
on the combat-free lounge side of the door to Dave's Bar.

The `ConsignmentShelf` (`trade-shopkeeping/src/thing/ConsignmentShelf.ts`;
its mixin and surfaces stay in `lib/retail/Consignment.ts`) composes
`Persistable`, which is **load-bearing, not incidental**: it captures the
consigned goods + their `_chattelId`s into a durable record, so a consigned
player-owned good survives a relog while in the shop's custody (a transient
shelf would drop it).

## The distributor — the cash-and-carry, and consignment BY a business (libations)

The bar buys from a **distributor**, never from a `props:` line.
The **`distribution` pack** ships the distributor's MECHANISM
(fermentation D10 moved it out of trade-distilling so sibling trades
share no edges): `thing/counter` a `Stock` whose ONE stocked line is the
malt sack (the imported-input faucet) atop the consignment shelf,
`serverPositionKeys: [clerk]`; `idea/business` with `clerk` and `keeper`
(`purchases: true`); `agent/clerk` and `thing/racking`.

⭐ The **ROOM** is the locality's — `/world/terminus/counting-houses/cash-and-carry`,
a showroom with a roller door onto the avenue, described in that avenue's
own prose. Trade is mechanism and locality is expression, and the split
is what lets the room sit in a real grid: it plots at `(0,-1,0)` in The
Counting-Houses, one cell south of the block, which is also what makes
that cardinal exit pair legal. Everything
on the counter is **consigned by an authored consignor**: every producer
pack ships an *outfit* — a Business, a `Stock` its floor product stands
in, and a hand NPC running the kernel **`consigns`** brain whose config
names the HOST shelf (`{ stock, shelf, ask: {censusKey: minor},
defaultAsk?, batch? }`). The annex names the host; the distributor names
nobody. The hand holds a `purchases` position, carries the house card,
and each beat `get`s up to `batch` goods, teleports to the counter,
`wallet use house`, `consign <kw> --ask <n>` — as the business
(`ConsignController` accepts an `organization` owner the giver buys for;
the listing's `consignorKey` is the outfit's path, and "each consignor's
account rises on resale" is the shipped split leg). The floor itself
stands at target through the residency spawn sweep, the rows' own
`container:` naming the outfit's Stock ([residency.md](./residency.md)).

The buyer's side is the wallet rule ([employment.md](./employment.md)):
with the house account active, `buy` settles from it and stamps the
chattel to the business (the owner-principal builder widened: an
`OrganizationMixin` Stuff → the `organization` owner arm); with a
personal account it stamps the buyer, exactly as before. The `restocks`
brain is a keeper doing that loop on cadence.

**The stock vessels**: `/platform/thing/Bottle`
(`Circulating(Sealable(Detailed(GradedReceptacle)))` over `Thing`'s
Chattel — 0.75 L glass, liquid-tight; presets are ROWS: `keg`, `cask`,
`wine-bottle`, `can`, `mixer-bottle`, `sack`, `ice-bag`; distilling's
`SpiritBottle` extends it in code) and `/platform/thing/Crate`
(`Circulating(Staged(Container(Detailed(Thing))))`, open and never
Sealable so the crafting gather walk finds the limes; the fruit are
`Provision` rows its `props:` mints). A floor row authors `censusKey`,
`regionTarget`, `container: <the outfit's Stock>`, `interiorMaterial`,
`gradeBand`, `_brandKey`.

## Content: the store on the new-player path

Authored off the counting-houses avenue block
(`world-seed/content/world/terminus/general-store/`), NORTH from the street, on the
arrival→campus path. Its own zone + parcel (terminus-titled), addressed
under the counting-houses Locality. A shop floor `props` the counter,
the consignment shelf, and the cast (Pemby the clerk on the roster, Odell
Marrow the keeper/proprietor). Five **real** discrete staples — each backed
by a shipped system, not a decorative prop: a **torch** + **lantern**
(`PortableLight` = `LightSource` ⊕ `Switchable` — `switch` it on and it
genuinely casts light, off goes dark), a **waterskin** (`Receptacle`, a real
`Bulkable` fluid holder), a **clasp knife** (a real bladed `Weapon` —
wieldable, delivers an edge), and a **ration pack** (a `Thing` of an edible
`trail-ration` food material — `eat` routes its carbs to satiation). Priced a
coinage-clean ladder (2..10) against the 20-credit stipend. **Independent**
(no corpo — franchising is retail S4).

## Supplier terms, the stocking rule, and the rented stall (economic bootstrap, 2026-09)

> Full treatment: [credit.md](./credit.md).

- **A listing has a BASIS** — `ConsignmentListing.basis: consignment |
  terms` — decided by the counter's authored **`Stock.purchasing`**
  policy, never by the consignor. At a `terms` counter the `--ask` is the
  supplier's PRICE (owed at sale), the shop sets its own ask and keeps the
  margin, and a `buy` posts twice (the sale, then the shop's `terms` leg to
  its supplier). The cash-and-carry says `purchasing: terms`; so does every
  rented stall. `reclaim` admits the house an actor buys for, so a
  supplier's hand takes an unpaid crate back.
- **`StockLine.supplier` / `pricing`**: a line with a supplier is bought
  by the keeper's `stocks` beat and never cloned by `reset()`; a line
  without one is an import. `pricing: stocking` derives the ask from the
  shelf against par (`retail.stockingElasticity`); `Stock.priceFor` is
  the override, `basePriceFor` the authored number, `house price <thing>
  <ask>` the keeper's hand on it. A good the shop never priced asks the
  supplier's price × (1 + `retail.termsMargin`).
- **`look` at a counter good says the ask** — `Stock.termsLineFor`,
  appended by a `Chattel` markup augmenter: *held on the farm outfit's
  terms until sold; the shop asks eight zorkmids* / *on consignment for
  Alice at four zorkmids* / *the shop asks two zorkmids*.
- **The per-shelf cap**: `listingCapOverride` on a wholesale shelf (the
  cash-and-carry authors 120; the produce stalls 200) — and the
  `consigns` brain honors the shelf's own cap, as `consign` does.
- **A player shop is a rented stall** — `stall rent` / `stall give-up`
  on the market square (`MarketStalls`, the terminus pack): a counter
  minted with the renter's identity, a house re-minted from them, an
  account of their own. `Business.getAccountPath()` is the identity path.

## Deferred

The NPC buying goods for its own coin (the vendor-trash faucet, behind the
Circulation Reserve — retail S2); cost-plus / supply-demand pricing and the
producer/mine (retail S3); player-owned shops / franchising / the market
arena (retail S4); Branded-independent marks on the goods (cosmetic
follow-up); fungible-goods retail (with the broader glob-ownership
question).

A **coin toll for a service** (a `mill` service kind on `Tariff`): the
grain chain rejected it for the miller — a mill is paid in KIND (the
multure, a tenth of the grist into the toll bin), which is how it pays
for premises with no coin changing hands. If a venue ever wants coin for
a working, the `Tariff` vocabulary is the seam; nothing else needs
inventing.
