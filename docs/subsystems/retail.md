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

`/platform/thing/Tariff` = `PricedOfferMixin(Thing)` plus
`services: Record<offerKey, ServiceKind>` over a **closed kernel
vocabulary** — `repair` · `treatment` · `burial` · `transfusion` (the
blood window's product). `OrderController` resolves the key and collects;
the DISPATCH is kernel-owned but each **arm delegates to the subsystem
that owns the capability** and inlines nothing: `repair` →
`CraftingApi.repair`, `treatment` → `ConditionApi.treatWorstResolvable`,
`transfusion` → `DonationBankMixin.transfuseInto`, `burial` →
`Postmortem.interIn`. The closed vocabulary is why a venue is rows and not
pack code: a venue that could define its own service kinds would need
pack code, and then a second venue would too. ⭐ **A second clinic is a
`Tariff` row and a `Business` row** — the shipped proof is a repair tariff
at the hearthworks smithy and a second one, at a different price, at the
Rejection smelter (it stood in the provisioning shed until the repair
build — a shed with no fire and nobody whose trade a mend is).

### ⭐⭐ A priced service is performed by a PERSON (repair, 2026-10-17)

Until the repair build the `repair` arm mended as the CUSTOMER — at their
band, with nobody in the shop — and `collect` took the money: a priced
service with no worker in it. Now:

1. **The hand.** `CraftingApi.fulfillingHand(giver, tariff, discipline?)`
   — someone on shift in the room whose seat `fulfills` the work (any,
   when no discipline is named; a Tariff row may name one per service),
   working for the house the Tariff's money goes to (`tariff.house()`:
   the fixture's own operator first, else the room's — so a shared room
   never sends another house's hand), chosen by that house's `call` rule.
   ⛔ Nobody is a refusal in words — `NOBODY_TO_WORK`, *"There's nobody
   here to do the work."* — one sentence for every service, never a free
   service.
2. **The quote, before the money.** The hand looks it over
   (`CraftingApi.diagnose`, read-only — the same ladder the mend runs) and
   names each fault and its price: `Tariff.quoteFor(lines)` prices `{key,
   label, qty}` lines, a repair key the row does not price (`tighten`,
   `reforge`, `refire`) deriving from `repair` (×¼, ×2, ×1, grain), any
   other unpriced key staying unpriced (null — never derived). A fault the
   hand cannot mend here is said with its cure and not priced; a hand who
   can only make a part serviceable says so.
3. **Can you pay?** `PricedOfferMixin.canSettle(amount)` (`BankingApi
   .canSettle` — credential balance, else coin in hand) BEFORE the work: a
   mend nobody can pay for is refused, not done on the house.
4. **Yes or no.** `context.interactive.promptConfirm` (`prompt-confirm` on
   the wire); no interactive — a scripted caller — answers no.
5. **The mend is the hand's**: `CraftingApi.repair({item, hand})` — their
   reach, heat and bands; their name on the `MendRecord`.
6. **One settlement.** Every arm returns `{ok, detail, charge?, hand?}`;
   `doService` settles once — `collectMinor(charge)` for a computed quote,
   else `collect(key)` — and pays a `per-settlement` hand
   (`EmploymentApi.settlePiecework`, one piece) out of the house the fee
   went to; a time-wage hand is paid by the shift and not twice.

`PricedOfferMixin.collect(key, reason)` is the one settlement path for a
SERVICE (`collectMinor(amount, reason)` its computed-charge form), so a
`Menu` and a `Tariff` share it. A SALE (a stock good, a
consignment line, a priced menu recipe) settles through
`EmploymentApi.settleSale(venuePath, amount, taxable, reason, splits?)` —
the single path `buy` and `order` both run (it replaced a hand-rolled copy
in each controller). Income keys on the **Business** account (the same
account shift wages come out of); `ensureOperatorAt` stands the business
up lazily on the first sale; the venue's share-of-flow splits ride every
settle; credential first, then cash; the demo tax remits on the taxable
slice. ⚠ **Every failure is "on the house", never a throw** — no operator,
no bank, no funds: the customer is served and nothing is taken.

> ⛔⛔ **FINDING (drilling build, 2026-10-09): the failures are
> indistinguishable to the player, and one of them lies.**
> `settleSale` returns `null` for at least three different reasons — no
> venue path, no business operator, no operating account — and **all of
> them reach the buyer as `insufficient-funds`.** So a shop that *cannot
> trade* tells the customer their wallet is empty, which is the one
> explanation that sends them to fix the wrong thing.
>
> ⭐ The drilling drive supplies what the taps build could not: **a
> working comparison.** Six goods buy cleanly at the Terminus general
> store in the apiculture drive, so `settleSale` itself is sound and the
> defect is specific to Rejection's `provisioning-business`. A
> hypothesis worth chasing first: `operatingAccountOfImpl` takes an
> `openingAdvance` only when the account has no entries, and a sale
> remits the demo sales tax OUT of the shop — so **a cold shop at zero
> may be unable to remit the tax on its first sale**, which would make
> *the first sale a venue ever makes* the one that always fails. A
> bootstrap deadlock, invisible to any venue that has traded once.
>
> ⚠ Reported, not fixed: it is this subsystem's, not a trade's. The live
> evidence is the checkpoint
> `packages/wire/tests/drilling.dirty.wire.test.ts` § *the valley's own
> till cannot take money* — which asserts the DEFECT, so it will start
> failing when somebody fixes this, and that is the intended alarm.

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

- **`stockLines`** — authored `{ itemTemplatePath, par, brandKey?, from?,
  pricing?, buys? }`; the offer key is the item template path.
- ⭐ **The shelf is always instances; a line says where it is REFILLED
  from — exactly one source, `from:`** (storage-and-ice review). Absent: an
  **import**, cloned from nothing at the sweep. `{ supplier: <business> }`:
  **bought**, by the keeper's `stocks` beat, on terms. `{ store: <store> }`:
  **brought up** from a store (`lib/storage`) on the house's own claim, at
  the sweep and to order. `setStockLines` refuses a line naming two sources
  (and the retired `supplier:` / `source:` keys); `Stock.sourceOf` is the
  one place the shape is decoded. ⚠ The source is per LINE, never per
  counter class: one general store may carry imported, supplied and
  store-fed lines on the same counter, and a second venue is rows.
- **`reset()`** (the `ResettableMixin` hook) tops each import line back to
  par by cloning fresh goods from `itemTemplatePath` — **items, never
  money** (Law 2) — skips a supplied line, and brings a store-fed line up.
  `resetsWhilePresent() → true` (restock while browsed is fine).
- **One matcher** — `Stock.lineForWord(word)` is how a typed word names a
  line (the template leaf, or one word of it: `block` for `ice-block`);
  `buy`'s sold-out refusal and the bring-up both ask it.
- **Self-stocks on standup**: `onCreate` calls `reset()`, so the same
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
`EmploymentApi.ensureOperatorAt`, inside the shared
`EmploymentApi.settleSale`.

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

⭐ **And `resolveBuy` answers to the same leaf** (`Stock.resolveBuy`,
keywords first, the leaf the fallback — matched as `carriesLine` matches
it). Before the rubber build's browser drive (2026-10-18), `buy
rubber-boots` — the word the shelf list prints, `rubber-boots (11)` — was
refused as *the shelf is bare … none of it today* while a pair sat on the
counter.

⭐ **A bought good is placed in the buyer's estate** (fishing B7,
2026-09: a bought rod vanished at the next restart). Since the assembly
build the movement primitive records it — `buy` stamps the title (a sale
is a title act) BEFORE handing over, and `move` does the rest. ⚠ `buy`
sells ONE piece off a stocked heap at the piece price, and a counter must
be one of its house's `operatingLocations` (`lint:shopfronts`). The fish stall (`market/thing/fish-stall.yaml`) is a second
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

⚠ **A relog, not a restart — unless the counter establishes its own
record** (found by the repair drive, 2026-10-17). A `props:` counter like
the general store's is cloned fresh at every boot and has no persistence
key, so it never captures at shutdown and never materializes — and the
owner overlay that would put a consigned good back runs only on
materialize. The good is not lost (its owner's estate keeps it, placed
at the counter); it is never placed. `MarketStalls.hydrationSource` is
the fix on the market square; whether every `Stock` should carry one
(a boot-time read per counter, and a keyless restore on a rented stall
before it is keyed) is open.

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
stands at par because the outfit's Stock NAMES its lines (`stockLines`,
made up by `Stock.reset`) — the goods say nothing about where they are
made (the scan sweep's review, 2026-10-15; until then each good's own
`container:` routed the residency spawn sweep's mints there).

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
(`Circulating(Staged(Container(Good)))`, open and never
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
- **`from: { supplier }` / `pricing`**: a supplied line is bought by the
  keeper's `stocks` beat and never cloned by `reset()`. `pricing: stocking`
  derives the ask as `base × (1 + e × scarcity)` (`retail.stockingElasticity`),
  where ⭐ **scarcity is asked of the line's source** (`Stock.scarcityOf`):
  a shelf line against its par (`1 − onHand/par`), a store-fed line against
  the HOUSE's days of supply in the store over `retail.stockingHorizonDays`
  (`Store.houseDaysOfSupply`: its claim paid pro rata, its share of the
  loss, its own draws — a depositor taking their goods home is not the
  house selling). `Stock.priceFor` is the override, `basePriceFor` the authored number, `house price <thing>
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
  keyed to the **pitch** it stands on, a house re-minted from the
  renter, an account of their own. `Business.getAccountPath()` is the
  identity path.

  ⭐⭐ **The key is the PITCH, not the renter (2026-10-05).** A key is
  relative to the thing that MANAGES it — a holding warren keys its
  rooms `<extent>/<leaf>` off its own durable address — and the square's
  fixture is what manages pitches. So the counter is `(scope = the stall
  seed row, key = <the fixture's row>/<pitch>)`, and **who rents it**
  lives on the house's `appointingAuthority`, where it belongs. The
  fixture keeps the book of lets (`pitches` authored, `lets` runtime),
  allocates lowest-free, and `stall rent` refuses a full square before
  any money moves.

  Keying on the renter was wrong three ways: it made the renter's
  identity the counter's substance; it let a square hand out unbounded
  stalls, because nothing counted pitches; and it put the counter in a
  population the `(scope, key)` uniqueness invariant **could not scan at
  all** — the scan's needle was the host's own identity, so for an
  identity-stamped keyed host it read a bucket holding one object and
  passed (see [persistence.md](./persistence.md)).

  ⚠⚠ Two things the re-key forced, both of them the same clobber from
  opposite sides:

  - **The fixture must establish its own record.** A `props:` fixture on
    a non-`Persistable` street is established by nobody —
    `Stock.onCreate` only resets, and `capturesAtShutdown()` is false
    while the persistence key is null — so the book would come up empty
    at every boot and the next renter would be allocated an occupied
    pitch. `MarketStalls.onCreate` materializes its keyless record.
  - **`give-up` DELETES the pitch's record** rather than capturing the
    emptied counter. A pitch-keyed record that outlives its let hands
    the next renter the previous keeper's counter, rent apparently
    already paid. A given-up pitch remembers nothing.

## ⭐ The house buys — `sell <thing>` (storage-and-ice review)

The counter's act, `Stock.buyFrom(seller, item)`, returning its refusal in
its own words; `SellController` is dumb. A line says the house buys it —
`buys: true` (an ice company buys a cutter's blocks; a general store does
not take your limes). ⭐ **The bid is derived, never authored**:
`Stock.bidFor` = the ask ÷ (1 + `retail.termsMargin`) — the terms rung's
markup run backwards — so a house that is short both asks and PAYS more,
which is the price a harvester reads. Every gate is asked before money
moves: a line it buys; somebody behind the counter; the good is the
seller's own; a store-fed line's store will take it (its acceptance list —
an icehouse takes `ice` and `sawdust`, a pool KIND or one key); the seller
has an account; the house can pay. Then `BankingApi.payTerms` (the house
paying a supplier: cost of goods on its books), and the good goes where
the line is refilled from — into the store on the house's claim, or onto
the shelf as the house's.

⭐ **The counter says what the house buys before anybody makes the trip**
(a `look` at the counter): the bid; or *"its store is full"* (the store's
`capacityKg`); or *"it is short of money"* — remembered from the last sale
that failed on the house's account and cleared by the next good one,
because a look cannot read a balance. A buying line's store-fed ask is
the counter's own (`bringUpFor` brings a block up **to order** when the
shelf is bare, so an empty store is an empty counter).

⚠ **Many sellers, one counter** is held by the market, not a cap: the bid
falls with the house's days of supply, and its store and its money run
out. Exclusivity of a harvest (who may cut a pond) is a use of the reach
→ navigable-water-slate § Ice
harvest as a use of the reach.

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

## Blood-economy build — the `transfusion` service + two houses in one room

`SERVICE_KINDS` grew a fourth kernel-orchestrated service, `transfusion`:
`OrderController.performService` draws a compatible unit off a
`DonationBankMixin` fixture (the blood window) and `receiveBlood`s the
customer (customer = patient, the only shape the payer rule allows). Two
kernel fixes let two houses share one room: (a) `PricedOffer.collect`
resolves the operator **self-first** — the priced fixture's own operating
business before its container — so a blood window's fee attributes to the
window's house, not the practice sharing the ward; (b) `OrderController`
**scans every reachable `Tariff`** when the bound counter does not price
the key, before falling through to the menu. See `blood.md`,
`employment.md`.

## ⭐ A priced service performed by a person — wash and bath (2026-10-17)

The cleanliness build added two service kinds, `wash` and `bath`, on the
seam a priced service is performed by a person: `CraftingApi.fulfillingHand`
(restricted to the tariff's own house, and to `Tariff.serviceDisciplines`
when the row names one), `Tariff.quoteFor`, `PricedOffer.canSettle` before
the act, one collector, and piece-rate pay by the seat's basis. A ruined
wash is paid back by the house's posted `cover` through
`BankingApi.indemnify` — never to one of the house's own seats. `bath` is
durative on the customer. See [cleanliness.md](./cleanliness.md).

## Note — the potash build (2026-10-19)

Three more buying lines: the Wharfside chandlery buys wood ash, black salts
and pearl ash; the Hanging Wood glasshouse's chalked board buys pearl ash
and black salts. The ash round is a sole trader who buys at the hearth at
the chandlery's own bid and sells at its counter. See
[potash.md](potash.md).
