# Credit — the ladder, the Note, and the estate

> **Status:** built by the economic bootstrap (`build/economic-bootstrap`,
> 2026-09-19/20). This doc is the source of truth for the credit substrate:
> how money ENTERS the world (two lanes and nothing else), how a shop or a
> person borrows it (a ladder of ledger-gated rungs), what a newcomer
> arrives owing (the Arrival Note), and what happens to everything a
> member holds when they stop coming (the three estate states). The
> money mechanics themselves — legs, the floor, conservation — are
> [banking.md](./banking.md); the contract rows the paper lives in are
> [contract.md](./contract.md). This doc is the part that is *finance*.

## The one sentence

**Nothing in this world mints money by authoring it.** `lint:no-authored-
faucet` holds the count at zero. Money enters by two rules a member can
read in the Schedule and predict — the **window** (lane one) and the
**perpetual** (lane two) — and every other zorkmid anybody ever holds
was lent, earned, sold for, or advanced against something.

## The two lanes

| lane | what moves | the rule | who can see it |
|---|---|---|---|
| **one · the window** | `mint` legs, category `window`, ISSUANCE → a chartered bank; `drain` legs back as the bank repays | a bank presents inventory paper (a rung-1 loan) at the window and is advanced its face, at `reserve.windowRatePerYear`, less `reserve.haircut`; every repayment split flows back and the window is repaid pro rata | `reserve` — *window advances outstanding* |
| **two · the perpetual** | one `mint` leg, category `perpetual`, ISSUANCE → the treasury | `reserve.moneyPerActiveMember × PlayerApi.activeMemberCount()`; the reserve buys the state's perpetual up to that and **never redeems**; runs on every treasury touch | `reserve` — *the perpetual, held* |

A third hand-typed number exists — `reserve override <n> to <who>
"<why>"` — and it is **recorded**: category `override`, the memo on the
ledger, the total on the dashboard forever. It is the Governor saying
"this once", in writing, and the wire suites fund themselves with it so
that the funding is on the record too.

The **treasury** is `/compact/treasury`'s account at the Central Bank.
The Minister of Finance **appropriates** from it (`treasury appropriate
<n> to <house>`, a real → real `appropriation` leg — the supply never
moves); the state never spends what the rule has not bought.

⭐ A lane's outstanding is a **warmed read**, not a scan: `SupplyAggregate`
carries the same minted/drained sums PER CATEGORY the headline supply
carries in total, kept by the same post. Before that, every treasury
touch and every `reserve` summed the ledger (7.5 s after fifteen minutes
of keeper beats). The ledger is the truth; the lanes are its running
total, rebuilt by `recomputeSupply` like the headline.

## The floor

Every `postTransaction` projects each account's net against its cached
balance before it writes: **no leg may drive a real account negative.**
There is no overdraft anywhere in the substrate — a wage the house cannot
cover is a working-capital draw (rung 2) or a refusal with an arrears
line, never a red balance against no creditor. See [banking.md § THE
FLOOR](./banking.md).

## The ladder

A **shop** climbs it; a **person** climbs it the moment they keep a shop
(a rented stall is one). Every gate is a **read of the borrower's own
ledger** — nobody judges.

| rung | who lends | secured by | the gate | the verb |
|---|---|---|---|---|
| **0 · supplier terms** | the supplier | the goods — title stays the supplier's until sale | none | `consign … --ask <price>` at a counter whose `purchasing` is `terms` |
| **1 · inventory finance** | a chartered bank, funded at the window | the goods, with the bank's lien on the pledged counter | `reserve.ladder.termsRequired` **completed purchases**, no defaulted loan *still owed* | `bank borrow <n> --for stock` |
| **2 · working capital** | a chartered bank, from its own balance | unsecured, capped at `reserve.ladder.workingCapitalCap` | `reserve.ladder.loansRequired` rung-1 loans repaid, no default still owed | `bank borrow <n> --for wages`, or the wage seam's automatic draw |

### Rung 0 is consignment grown up

`Stock.purchasing` (`consignment` | `terms`, the counter's authored
policy) decides what a `consign` at that counter MEANS. At a `terms`
counter the consignor's `--ask` is the **price** — what they are owed at
sale — and the **shop sets the ask** (`Stock.priceFor`: a `stocking`
line's rule, or `house price <thing> <ask>`, or the supplier's price ×
(1 + `retail.termsMargin`) until the house prices it). A sale posts
**twice**, one command: the buyer pays the shop's ask (`sales`), then
the shop pays its supplier (`payment`, category `terms`). `look` at the
crate says whose it still is — *held on the farm outfit's terms until
sold; the shop asks eight zorkmids* — and `reclaim` lets the supplier's
hand take an unpaid crate back off a shop's counter: rung 0's
repossession is a query over who owns it. `house book` lists the terms
payable per supplier (Σ over unsold terms listings): nothing is owed
until sale.

### Rung 1 reads the purchase history

⭐ *"A shop with a clean purchase history"* — the gate counts `payment`
legs OUT of the borrower's account that paid a supplier for goods, in
both the shapes that event takes: a `terms` leg (rung 0 completed at
sale) and a `sales` leg to another house's account (a keeper paying at a
supplier's counter, the same term settled on the spot). The refusal
names the number: *three completed supplier terms are required; you have
two*. The loan is a `loan` row (issuer = the borrower, holder = the
lender, `terms.rung: 1`, `security: inventory` on the pledged counter);
the bank presents it at the window and is advanced its face.

### Repayment is a share of inflow

Every settlement to a borrower's account carries its creditors' splits —
`ContractApi.repaymentSplitsFor` appends them to the `Charge` inside the
same conserving transaction, `repaymentShare` of the inflow (the bank's
own Terms, clamped by `reserve.repaymentShareMin/Max`) — and the row
accrues `(1 + r)^Δyears` stamp-forward (`GAME_YEAR_S = 360 × 86400`; a
game-year is a real month at the shipped scale). No due date, ever: the
row settles when the balance clears, the lien releases, the window is
repaid pro rata.

### Default is revealed, never scheduled

A loan whose borrower has had **no inflows for
`reserve.defaultHorizonGameDays`** while a balance stands is in default
the moment anyone reads it (`ContractApi.reconcileLoans`): the row flips
to `breached`, the creditor's rule acts — every good on the pledged
counter the borrower owns goes to the lender's own counter — and the
window advance behind it is written off. The reserve's **default rate**
(`reserve` prints it) is the inflation dial the Governor watches:
window paper that defaulted over window paper advanced.

### ⭐⭐ The record is permanent; the exclusion is not

A breached row **stays breached for good**. The default happened, the
papers keep saying so, and that record is what a credit history *is* —
it is also exactly what a future rating needs, because the two facts a
rating turns on are both on the row: that it defaulted, and whether it
has since cleared.

What lifts is the **bar**. `defaultedLoansOf` counts breached loans that
are **still owed**, so:

- the creditor's claim on the borrower's inflows **continues past the
  default** — the security did not cover the debt and the lender is
  still out of pocket, so the same share keeps coming out of every sale;
- **interest stops at the breach.** The shortfall is a fixed sum to
  clear. A debt that kept compounding would outrun the borrower and the
  lift would be a lift in name only;
- when it reaches zero the row is marked **`satisfied`** — an event, not
  a state change. It is never `settled`: *settled* would erase the
  default, and nothing here erases a deed;
- the reserve's default rate counts what the window is **still** out of
  pocket, so a cured default stops feeding a policy dial with a loss the
  reserve has already got back.

⚠ It shipped the other way for one review round: nothing could repay a
breached row (`recordRepayment` returned early unless the state was
`open`) and nothing cleared the count, so a single default ended a
business's access to credit **permanently, with no way to pay its way
back**. Nobody chose that — it fell out of the gate being a bare count
of breaches, which is how the harshest rule in a system arrives without
an author. **A refusal is only honest if something lifts it.**

### Rates are Terms

A bank that lends posts its rate beside its fees: `Terms.loanRatePerGameYear`
and `Terms.repaymentShare` on its `BankCounter`. A counter with no rate
does not lend. The banking hall's rate board prints the offer in words —
*FIVE PER CENT A GAME-YEAR (A REAL MONTH)*. There is no benchmark object.

## The Arrival Note

A newcomer signs no form and meets no clerk: at `embody confirm` the
machine writes a `note` row (issuer = the member, holder = the treasury,
principal `treasury.arrivalPrincipal`, **rate none**, discharge on the
first wage or after `treasury.noteDischargeGameDays` active, secured by
the balance it funded and **nothing else** — no recourse beyond the
security, no labor ever owed), disburses the principal into their hands
as coin (`disburse` — a withdrawal from the treasury, not a mint), and
files the paper at `/home/<key>/papers/arrival-note` where `cat` reads
it in **words**, no digits. `wallet` lists it beside the balance. The
first wage (`EmploymentApi.payHouseWage` → `ContractApi.onWageLanded`)
discharges it with a message; a touch past the game-days does the same
lazily.

⭐ The former **stipend** — a `mint` at character creation — is gone with
the faucet lint. Every arrival is a loan the arrival never has to repay,
which is the same twenty zorkmids with a different sentence attached, and
the sentence is the point: *no labor is ever owed*.

The **standing facility** is the Note's sibling for a house: a business's
first operating account is advanced `treasury.openingAdvance` at no rate
(a `loan` row, `rung: opening`), repaid as a share of inflows. ⭐ It is a
**float, not capital** — fifty, less than one restocking beat — so
inventory is financed where the ladder puts it. At two thousand no shop
in the world ever borrowed and the ladder was theater.

## A wage is a loan or a refusal

`EmploymentApi.payHouseWage(business, workerKey, amount)` is the one
seam a house pays a wage through: arrears first; then a rung-2 draw at
the house's bank where its ledger has earned one; else the wage is
**refused**, recorded as an arrear with the worker as creditor by name,
and the proprietor is told why. `house book` lists the arrears; `house
pnl` prints them. Nothing is paid into the red.

## The stocking rule — the NPC shop that keeps borrowing

A `StockLine` with a `supplier` (a Business path) is **bought, never
cloned**: the reset sweep skips it, and the keeper's `stocks` beat
(`lib/behavior/stocks.ts`) walks to the supplier's counter each cadence
for every supplied line short of par, `wallet use house`, `buy`s the
shortfall (by a keyword that names the GOOD, not the first crate), and —
before buying, when the house holds less than the beat costs — walks to
the house's bank and `bank borrow <shortfall> --for stock`. A refusal at
the window is the ladder working. A line with no supplier is an
**import**: goods from nothing, never money (the malt sack nobody
produces). `pricing: stocking` on a line makes its ask the market-maker's
— `round(base × (1 + e × (1 − onHand/par)))`, `e = retail.stockingElasticity`:
an empty shelf asks half again the base, a full one the base, an
overfull one less. An offer, never an oracle — and the **price index**
reads it.

Every step is the literal verb through `forceCommand`, on the real road
— the keeper walks the **authored directions** on her own row (`ways`,
the `patrols` shape: her shop is on the block, the window one west and
the wholesaler one south) rather than planning a route, because a search
over the freight network to cross the road is not one — with the house
card dealt at hire. The general store's keeper is the shipped exemplar: the
shop floor and the business are `boot:` producers so she stocks with
nobody online; her purchase history is the ladder's first climb.

## A player shop is a rented stall

`stall rent` on the market square mints, for the renter, a counter
(`/world/terminus/market/thing/stall`, cloned with the renter's IDENTITY
and persisted under their key — `purchasing: terms` from the seed) and a
house (`/platform/idea/Business/stall`, re-minted from the renter every
time: the authority is them, the bank is theirs, the counter is the one
it operates). `Business.getAccountPath()` reads the identity path, so
two renters are two accounts. The first rent is a `transfer` to the
market's business; `stall give-up` hands back the goods and takes the
counter down. Minimum that makes rungs 0–2 reachable by a player;
franchising and frontage stay with the retail slate.

## The index and the Gazette

`BankingApi.priceIndex()` reads the counters `reserve.indexBasket` names
and, for each `stocking` line, divides the ask by the authored base; the
index is the mean, printed as a whole percentage of base (a hundred is
par). `reserve` prints it to the Governor. **The state aggregates, never
reports**: the Terminus Gazette (`/world/terminus/gazette`, a public
publisher on the committee's chart) prints it — its editor's `prints`
beat runs `press post "Prices: the basket stands at one hundred and four
against a base of one hundred" --as /world/terminus/gazette --kind
notice` once an edition window of GAME time (`press.indexEditionGameHours`).
Journalism a member can walk the row and check.

## The three estate states

Derived from **absence** and nothing else. `PersistedRecord.writtenAt`
is stamped on every capture; for an Avatar the last capture is its
logout, so the row's clock IS "last seen" without opening the opaque
state. `PlayerApi.estateStateOf(identityPath)`:

| state | when | what it does |
|---|---|---|
| **active** | connected, or the row inside `estate.dormantAfterDays` | nothing |
| **dormant** | absent past the short clock | the account is **frozen** — outflows refused with the reason (withdraw, transfer, draw, a settled charge's payer), credits still land; a seat held is **vacated** (`vacated`, a terminal exit the roster never resurrects; the house card leaves the wallet); a shop kept shows the **closed sign** (`Business.isClosed()`, the counter's `requestAttention` answers `closed`, no shifts run); a login lifts it — what dormancy did stays done |
| **escheated** | absent past `estate.escheatAfterDays` | on the next touch the estate **passes** (below); the treasury holds the balance as **unclaimed property**, a claim the state cannot refuse |

`PlayerApi.activeMemberCount()` — the perpetual rule's denominator — is
the connected set ∪ every avatar row inside the short clock, one query.
**An NPC is never absent**: every read narrows on an Avatar identity
before it consults the snapshot.

### The touch

`PlayerApi.touchEstate(key)` runs at login (`returning: true`), at every
credit landing on a member's primary (a wage, a settled charge's payee
and splits, an escrow release, an appropriation), and at the roster pass
for every member who runs a house (`EmploymentApi.bringCurrent`, which
also vacates absent holders and writes the closed sign — the roster tick
runs it for every house, `buy` and `house roster` for one). At a return:
the reclaim (below), or the vacancy when the row's clock passed
`employment.absenceVacatesAfterDays`. At any other moment: the escheat,
once, past the long clock.

### The escheat — debts first, then situs up the title tree

`PlayerApi.escheat(key)`, in order, on the avatar stood up for the act
(the login clone with no connection; destructed after):

1. every loan reconciled — a default revealed repossesses;
2. the open Note **recovered** from the balance that secured it,
   `min(balance, owed)`, a `recovery` leg, the row settled whatever was
   there — non-recourse means exactly this;
3. player-held titles **transfer** to the parent parcel's owner (or to
   the named **beneficiary**, `wallet beneficiary <player>`, if they are
   themselves active);
4. use-grants revoked — the dorm, a let unit — and the enclosing zone's
   `onUseGrantRevoked` hook fired (a no-op terminal; a landlord zone
   overrides it to do what its own end-of-lease verb does);
5. a house they are the entity of: its counters retired (the goods to
   the room's own shelf), its account passing like theirs;
6. the balance to the treasury by an `escheat` leg, and either on to the
   beneficiary (a `repayment`/`unclaimed` leg) or an **`unclaimed`**
   contract row written — issuer the treasury, holder the member,
   reclaimable forever;
7. `Avatar.escheatedAt` stamped — the idempotence marker.

Chattel lying in the world is finders-keepers. Pets and livestock are
not escheated by this build (the pound keeper is the institutions
slate's; a bonded animal's transfer needs the pets consent model).

### The reclaim

On login, an avatar stamped `escheatedAt` has every `unclaimed` row it
holds paid by the treasury to its primary (`BankingApi.reclaim` —
the perpetual reconciles first, so the floor is met; the treasury cannot
refuse), the stamp cleared, and a scene that says the treasury paid.

## The Schedule (the rows this subsystem reads)

`reserve.*` — `moneyPerActiveMember`, `windowRatePerYear`, `haircut`,
`ladder.termsRequired`, `ladder.loansRequired`, `ladder.workingCapitalCap`,
`repaymentShareMin/Max`, `defaultHorizonGameDays`, `indexBasket` (the
Governor's, via `reserve set` — and only these). `treasury.*` —
`arrivalPrincipal`, `noteDischargeGameDays`, `openingAdvance`.
`estate.dormantAfterDays`, `estate.escheatAfterDays`,
`employment.absenceVacatesAfterDays`. `retail.stockingElasticity`,
`retail.termsMargin`. `press.indexEditionGameHours`. Every one lives in
`packages/content/platform/content/settings/*.yaml`; every one is read
by `AppApi.setting` with a code floor that equals the shipped value.

## The two lints

- `lint:committees-are-players` (ceiling 0) — no NPC on a committee; a
  committee is title holders, players only, by arithmetic.
- `lint:no-authored-faucet` (ceiling 0) — no `mint` outside
  `BankingLogic`'s two rules, the recorded override and the coinage
  bridge; the retired stipend key stays refused by name.

## The Discipline

`finance` (ISCED-F 0412, under `business-admin-law`) is credited on the
giver at every act that puts their name on paper: signing the Note,
`bank borrow` (granted or refused — the refusal is the lesson), reading
a book (`bank book`, `house book`), `wallet beneficiary`. The NPC path
credits nobody; the rule's act credits nobody.

## Deferred seams

- **The dorm's landlord override** of `Zone.onUseGrantRevoked`: the
  grant's revocation alone ends the tenancy (the door refuses the key);
  the room reverts at Katie's `unprovision`. The hook is on `Zone`; the
  dorm is a `Warren`.
- **The beneficiary chain**: a non-active beneficiary is skipped and the
  state holds the estate; chaining onward to their beneficiary is a
  refinement.
- **A stall at boot**: nothing pins a rented counter; it stands from the
  keeper's next `stall`. The closed sign covers the absent keeper's case.
- **Pets, livestock, chattel lying in the world** — see the escheat.
- **Rung 3, the debt market** — the auction slate's.
