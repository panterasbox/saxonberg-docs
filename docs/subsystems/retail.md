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

The ownable staples **must be discrete `Thing`s, never `Globbable`** —
chattel stamps a discrete instance, so a fungible stack would fall through
the buy/consign loops (a CI-adjacent content-integrity lint enforces this
on the seeds).

## The buy loop (`buy <thing>`, stamps ownership)

`BuyController` (`obj/command/retail/`, the new `retail` command category)
resolves the good off the present `Stock` **or** a consignment listing on
the `ConsignmentShelf`, settles a presented `Charge` (card → cash), hands
the good over, and transfers ownership:

- **stock good** — settle the full price to the store's Business account,
  remit the demo sales tax on it, and **stamp** the buyer
  (`ChattelApi.stamp` — a fresh, author-owned shelf good changes hands).
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
  an ownership pointer; "whose is it" is always `ChattelApi.ownerOf`.
  Gates: you must own it (`ownerOf`, not custody); it must be discrete (a
  glob is refused); you must hold a bank account (the payout target —
  nudge to Goodkin otherwise); and you must be under the **per-consignor
  listing cap** (`retail.consignment.listingCap` — the shared-shelf
  anti-grief guard, the withdrawal-quota sibling). An unstamped
  author-owned good is stamped to you on consign (establish-on-consign).
- **A sale** splits the ask (commission → store, remainder → the
  consignor's **primary bank account** via `Charge.splits`), transfers
  custody **and** the owner-stamp to the buyer, and clears the listing.
  The store **fronts no coin** (a real buyer's coin funds both legs);
  conservation holds (`reconcile().balanced`). The consignor need not be
  online — payout rides `primaryAccountIdOf`, a pure DB read.
  `commission = ask × retail.consignment.commissionRate`.
- **`reclaim <thing>`** returns custody of an unsold listing — **no
  chattel op** (ownership never left you).

The `ConsignmentShelf` (`lib/retail/ConsignmentShelf.ts`) composes
`Persistable`, which is **load-bearing, not incidental**: it captures the
consigned goods + their `_chattelId`s into a durable record, so a consigned
player-owned good survives a relog while in the shop's custody (a transient
shelf would drop it).

## Content: the store on the new-player path

Authored off the counting-houses avenue block
(`seeds/domain/terminus/general-store/`), NORTH from the street, on the
arrival→campus path. Its own zone + parcel (terminus-titled), addressed
under the counting-houses Locality. A shop floor `populates` the counter,
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
