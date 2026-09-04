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

## The shared price-list (`PricedOfferMixin`)

Extracted from the bar's `Menu` so the bar and the store share one offer
model (bar parity a hard constraint — the bar's tests are the gate).
`PricedOfferMixin` (`lib/commerce/PricedOffer.ts`) owns **only** the price
map (`prices: Record<offerKey, minorUnits>` + `priceFor` / `setPrice` /
`pricedKeys`) — Law 1: worth lives on the *offer*, never on the good. What
an offer key *means* stays the consumer's concern (a recipe id for the
bar's `Menu`, an item template path for the store's `Stock`).

## The counter (`Stock`)

`lib/retail/Stock.ts` — one fixture that is the whole counter: a `Vessel`
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

The ownable staples **must be discrete `Thing`s, never `Globbable`** —
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
base**. `CheckRack` (`platform/thing/CheckRack.ts`) = `Persistable +
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

The `ConsignmentShelf` (`lib/retail/ConsignmentShelf.ts`) composes
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
(`Circulating(Populates(Container(Detailed(Thing))))`, open and never
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

## Deferred

The NPC buying goods for its own coin (the vendor-trash faucet, behind the
Circulation Reserve — retail S2); cost-plus / supply-demand pricing and the
producer/mine (retail S3); player-owned shops / franchising / the market
arena (retail S4); Branded-independent marks on the goods (cosmetic
follow-up); fungible-goods retail (with the broader glob-ownership
question).
