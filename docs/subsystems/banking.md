# Banking — the monetary substrate

> **Status:** built on `feature/banking-build` (phases 1–5). This doc is the
> source of truth for the banking substrate. Sections marked *(deferred)* name
> parked seams.

Banking is **phase 4 ("Money") of the Dave's-Bar track**: the two-tier money
model (physical **cash** vs auditable **account balances**), bridged by a
custodial **bank**, with a single governed-but-logged **central bank** mint.
It mirrors the `lib/standing/` **append-only-log → rebuildable-materialized-
standing** shape (renown / participation / producer) with one hard addition:
**conservation**.

## The money model (the load-bearing decisions)

Money exists in two domains, joined only at the bank:

- **Cash** — `Coin`, a physical `Globbable` `Thing` carrying per-coin
  **mass**. Off the *governed* ledger: a hand-to-hand handover changes
  *location*, not *total supply*. Self-limiting by mass (a `Coin` stack's
  `getMass()` is per-coin × quantity, read by the shipped `LoadBearing`
  gauge — the cap on cash is honest physics, not a rule).
- **Account balances** — weightless minor-unit balances **derived from an
  append-only ledger** (`bank_ledger`), materialized in a warm cache
  (`bank_accounts`), behind the gated `BankingApi` / `BankingLogic` pair.

`Money` is the amount value-object: integer **minor units** + a currency tag.
It is a *transient settlement quantity* and a *ledger record* — **never a
"worth N" stamped on a good** (Law 1). ⚠ The currency argument is
**required, never defaulted**: a default lets a call site silently assume the
wrong currency, caught (if at all) only at runtime, where a required
parameter turns the whole threading into a compile-error list.

**v1 registers exactly one currency: the ZORKMID.** `credit` was retired
with the currency build — the word belongs to the deferred lending subsystem
(tabs, interest, creditworthiness, insolvency) and to the credit/debit
vocabulary every ledger leg already speaks.

### Currency — the record, and the one place it lives

`lib/banking/Currency.ts` is a **registry** (not a data Idea + catalogue)
holding every currency-intrinsic fact: the denomination table, the render
vocabulary, and the issuer.

```ts
interface CurrencyRecord {
  key: string;            // 'zorkmid' — the durable join
  unit: string;           // render, singular
  plural: string;         // render, plural
  issuer: string;         // the institution that mints it
  governorOffice: string; // whose holder may mint it
  denominations: readonly { value: number; massKg: number; label?: string }[];
}
```

A registry because `Currency.faceValueOf` sits on the hot path of
`Coin.getMass()` and the scope-walk keyword getter and **throws** rather
than guessing — a catalogue warmed asynchronously from `domain` would
resolve nothing in every test that skips the clone pipeline. It is also why
the test-only fixture currency can never reach a live collection: currency
records are *code*, and code is not a collection. **Adding a currency is a
code edit at the wizard tier** — the reserved-matter constraint in its
crudest honest form (a mint is Compact-level, never a locality's own call).

> ⚠⚠ **The record carries no rate, and no reference to another currency —
> permanently.** A `pegRate` field beside the denominations is the
> *world-oracle* shape: every reader of the currency would get an
> authoritative cross-currency rate for free, which makes a price a
> *property of a thing* rather than *an event between two parties*
> (economy-slate Law 1). A **peg is an issuer's standing offer to redeem** —
> reserves plus a published rate honoured at its own window, breakable when
> the reserves drain — and it belongs to that offer, not to the money.
> A test asserts the record's shape carries no such field.

**Denomination identity is `(currency, faceValue)`** — structural, not an
authored name. `zorkmid` is the only authored money noun; a coin presents as
*"a 25-zorkmid piece"*, derived. The optional `label` is for an issuer that
wants named coins (a corpo scrip will; the Compact does not). The retired
`sovereign` / `crown` names had never been seen by a player: one coin
template, and `issueCash` restamped only the denomination, so every coin read
*"a credit coin"* and answered to `credit`.

**Which currency?** `banking.compactCurrency` (an AppSetting, read only
through `Currency.compact()`) names the one the Compact transacts in. That
is **policy data, not a property of the money** — it is where the zorkmid's
specialness lives, so no code path compares a currency to a literal. Reserve
status is functional, never decreed: make Compact obligations payable only
in zorkmids and it is the reserve *by construction*.

### Conservation

> **Total money supply** = `Σ(mint amounts) − Σ(drain amounts)` over the
> central-bank log, **per currency**. It changes **only** by a mint (faucet)
> or drain (sink). Every other posting conserves it.

Conservation is **N independent domains**, one per currency — `bank_supply`
holds one row each, and every leg carries the currency it moves.

> ⚠⚠ **A ledger leg may never cross currencies.** `from` and `to` are always
> denominated the same. There is no leg kind that converts, no flag that
> permits it, and **this is a permanent invariant, not a seam awaiting an
> exchange**: currencies are *goods*, bought and sold in the market that
> already exists at whatever price people pay. Nothing in this codebase asks
> what one currency is worth in another, and nothing ever should. Relaxing it
> would create money in one currency and destroy it in another — an invisible
> mint, the one bug class an economy cannot recover from.

Two independent gates enforce it. `assertConserving` proves the legs agree
with **each other** (pure, no I/O). `postTransaction`'s **endpoint check**
proves they agree with the **accounts they touch** — reading the warmed
currency cache, never an await per endpoint — because a same-currency pair of
legs could still move zorkmids into a scrip account. An account the cache has
never seen is *new* (about to be created), so an empty currency adopts the
leg's rather than failing.

Enforcement is a **sealed posting chokepoint** — `postTransaction`, a
module-private free function in `BankingLogic`, the **only** code path that
writes a `LedgerEntry` or mutates an `AccountBalance`. Each ledger row is a
**transfer leg** (`amount` minor units, `fromAccount` → `toAccount`).
Conservation is *structural*, validated per leg by `BankTransaction`
(`lib/banking/Transaction.ts`) against the leg's `kind`:

| kind | from → to | supply effect |
|---|---|---|
| `mint` | issuance sentinel → real account | **+amount** |
| `drain` | real account → issuance sentinel | **−amount** |
| `deposit` | cash bridge → real account | neutral (coin → vault) |
| `withdraw` | real account → cash bridge | neutral (vault → coin) |
| `transfer`/`payment`/`wage`/`tax` | real account → real account | neutral |
| `escrow-hold`/`escrow-release`/`escrow-revert` | real account → real account (via the per-contract escrow account) | neutral |
| `draw` | real account → real account (business → proprietor) | neutral |

A leg of a non-mint/drain kind that names the **issuance sentinel** (money
from nowhere) **throws** — a programmatic conservation breach (the
crafting/containment discipline: contract violations are exceptions, not
boolean flags). The throw is pure (runs before persistence, DB or not).

The **sentinels** (`lib/banking/Account.ts`) are non-account counterparties
with no balance row: `Account.ISSUANCE` (mint source / drain sink — the only
counterparty that moves supply) and `Account.CASH_BRIDGE` (deposit source /
withdraw sink — the off-ledger↔on-ledger boundary). `Account.CENTRAL_BANK`
is a *real* account (it can hold/float money); "in circulation" excludes its
holdings.

The headline supply is kept O(1) by `SupplyAggregate` (`bank_supply`, a
single rebuildable row) and reconstructable by a full ledger scan
(`recomputeSupply`). The reconciliation invariant —
`supply == Σ account balances + Σ circulating coins` — is the conservation
audit (the coin term lands with the cash bridge in Phase 2+).

## Custodial bank ops (Phase 2)

A **bank** bridges cash and accounts 1:1 and can't fail. The capability is
`BankMixin` (`lib/banking/Bank.ts`), hosted on a **teller-counter `Thing`
fixture** (`BankCounter` = `BankMixin(ContainerMixin(Thing))`) *inside* the
branch Location — NOT on the Location, because a Location's own
`commandContributions` don't reach its occupants (only sibling *contents*
feed the `environment` bucket; the `Menu` precedent). The counter's contents
*are* the cash vault; `getTillLiquidity()` = Σ vault coin face-values.

- **Affiliation** rides a plain `corpoKey` resolved on read via `CorpoApi`
  (a bank is *affiliated to* a corpo, not a branded product). `openAccount`
  records that key on the account row — the readable affiliation edge.
- **Resolution by identity** — accounts key on `{owner, bank}` (owner =
  the context-derived `templatePath`, bank = the **institution key**).
  **The bank is an institution; a branch is a service point of it**: the
  five corpo banks are their corpos' finance arms, so a branch's key
  defaults to its `corpoKey` (`goodkin`/`hollis`/`vionne`/`aevex`/
  `veshko`; `BankMixin.getBank()`), and an independent bank authors its
  own key (a key with no corpo behind it — the `Brand.owner === ''`
  precedent). Your account **exists at the bank and is serviceable at
  every branch of it** — deposit/withdraw resolve the account by the
  branch's institution, while the **till stays per-branch** (the vault is
  physically here; liquidity≠solvency is a branch fact). No number is
  ever typed: `myAccountAt(bank)` resolves "your account at this bank";
  `primaryAccountIdOf(ownerKey)` is the receive-by-identity target; the
  first account an owner opens is their **primary**. Multi-account is
  native (per `{owner, bank}` — one account per institution, an account
  at each bank if you want).
- **deposit** moves the coin into the vault and credits the balance 1:1
  (`deposit` row, supply-neutral cash bridge). **withdraw** debits and hands
  out coin (split from the vault), bounded by **both** the balance (solvency)
  and the **till** (physical cash, AC#13 — a CB mint to a balance with no
  backing coin is what lets the till bind). **transfer** moves balance→balance
  (conserving), only from your *own* account (anti-spoof; actor from context).
- The **1:1 invariant** (vault coin value == Σ branch balances) holds across
  deposit / withdraw / same-bank transfer. Cross-bank settlement of physical
  cash is deferred (a cross-bank transfer moves ledger balances only).

The branch ops ride **one `bank` verb** with subcommands (the
`chat`/`alias` dispatch-on-`subcommand` precedent, not a verb-per-action):
bare `bank` → balance; `bank open` / `deposit <coins>` / `withdraw <amount>`
/ `transfer <amount> to <who>` / `balance`. `bank open` is idempotent, and
for an existing holder it **doubles as the wallet re-link**: the payment
credential is session-durable (a returning login's implant is
re-provisioned bare — see [credential.md](./credential.md)), so re-opening
re-links the held account onto the fresh credential instead of refusing. One `BankController` dispatcher
extends `BankingControllerBase` (`resolveBank` — the affording counter, else
the room scan; the crafting "agent performs, venue owns state" resolution).
The branch is authored as **city content**:
`seeds/domain/eternal/university-avenue/{bank,bank-counter,npc/teller}.yaml`,
one cell north of the arrival plaza (reachable from the born-with University
Avenue fast-travel node), affiliated to **Goodkin** (the retail bank).

## Uniform settlement + the credential ladder (Phase 3)

Settlement is **one primitive** — `BankingApi.settle(charge, method)` — across
every transaction kind and every method (the method is a *parameter*, the
`ContainmentApi.move` uniform-surface / polymorphic-internals pattern). The
thing owed is a `Charge` (`amount + payee + reason`, **presented** vs
**stated**); mechanism is polymorphic underneath:

- **cash** → coin handover off the governed ledger (coins split from the
  payer to `charge.payeeContainer`; no account touched; supply unchanged).
- **credential** → an on-ledger `payment` posting routed through the
  credential's account (with optional **remittance splits** — a cut to a
  third-party account alongside the main leg: payer −X, payee +(X−Σsplits),
  each split +its cut; conservation holds across all legs). `--from <corpo>`
  routes one payment from a specific linked account without disturbing the
  active setting. Returns a `SettlementReceipt` the scene reads.

The **credential** is now a `payment` **record** held in a
`CredentialWalletMixin` holder (the unified credential substrate — see
[credential.md](./credential.md)): the record carries the linked-account set +
active pointer + `spendCap` + `frozen` + `authorize`. The holder composes over
BOTH a `PaymentCard` (`= CredentialWalletMixin(Thing)`, a 1:1 bearer
instrument you can lose) and the born-with `CredentialWalletUpdate`
(`= CredentialWalletMixin(AetherHostedMixin(Idea))`, the one wallet app holding
every credential kind — installed once by `Avatar.installDefaultLoadout`,
body-bound). Reached via the MQL `reachable` pool filtered on
`MixinApi.isCredentialWallet` + a **non-frozen** `payment` record
(implant-first — the seed's on-person-first ordering puts the self-hosted
leg before carried cards), so a reissued card is found in place of a
revoked one. `openAccount` auto-links each new
account to the owner's wallet (first opened → active). `BankingApi`'s
credential surface (`activeCredential` / `setActiveAccount` /
`freezeCredential`) traffics in the `PaymentCredential` record; `issueCard`
returns the card holder (`Stuff & CredentialWallet`).

The **risk ladder + recourse**: cash = bearer, no recourse; implant =
body-bound (not a carryable Thing); card = bearer, bounded by `freeze`
(report-lost → `frozen`, account/balance untouched, reissue via `issueCard`)
and a per-credential `spendCap` (`authorize` refuses over-cap or frozen — a
security cap, never a fee). Verbs: `pay` (the one flat top-level settlement
verb — stated transfer; `--cash`/`--from`) + the `wallet` verb (bare → show
active; `wallet use <corpo>` switch; `wallet freeze <card>` report-lost +
reissue).

> **History.** Phase 3 shipped the payment credential as a
> banking-owned `PaymentCredentialMixin` over a `PaymentCard` Thing and a
> `PaymentImplantUpdate` aether twin. The **credential-wallet build**
> (`feature/credential-wallet-build`) folded it into the shared credential
> substrate: the behavior moved onto a `PaymentCredential` *record* held in a
> `CredentialWalletMixin`, the implant twin became the unified
> `CredentialWalletUpdate`, and resolution re-keyed to
> `MixinApi.isCredentialWallet`. The verb surface and the ledger are
> unchanged — only where the credential's state lives. See
> [credential.md](./credential.md).

## Tabs, wages, demo tax, the P&L (Phase 4)

- **Tabs** — `TabMixin` on the **venue `Location`** (the Bar): per-patron
  accrued unsettled charges, owned by the house so the tab outlives shift
  changes (the bartender acts on the venue's behalf). **Recognition-gated**
  via `RecognitionApi.recognizes(recognizer, patron)` (a new boolean read on
  the recognition surface) — a tab is a privilege of being *known*.
  **Skipping** is priced, not prevented: `skipTab` applies a `RegardApi`
  regard hit from the creditor and revokes the privilege; the unpaid balance
  stays on the books. State on the mixin (decision 3: session-durable). The
  `tab` verb (bare → show; `tab settle` / `tab skip`) is afforded by the
  bar's Menu (the affordance carrier in the room) and records against the
  venue's `TabMixin`.
- **Wages** — `BankingApi.payWage(employerAccount, workerKey, amount)` moves
  coin to the worker's primary account as a `wage`/`wages` line — the payment
  primitive only. *Who* is employed + *when they're paid* is now the
  **employment engine**: the roster tick settles a shift wage
  (`EmploymentApi.settleShiftWage`, `rate × shift-hours` at the shift-end
  boundary) from the **Business account**, skipping the proprietor's unpaid
  cover — see [employment.md](./employment.md). **No employer-solvency
  check** — the venue runs its P&L red by design (subsidy covers). `house
  payroll <worker> <amount>` (operator-gated via `AuthorMixin` +
  `requiresWizard`) stays the manual override, resolving the venue's account
  via the covering Business (`EmploymentApi.businessAt`), now redundant for
  the normal loop.
- **Demo sales tax** — `BankingApi.remitDemoTax(sellerAccount, saleAmount)`:
  a **seller-collected** `tax`/`tax` posting seller → placeholder treasury at
  the authored, **inert** rate (`banking.salesTaxRate` AppSetting; recorded,
  not governed — the corpo-affiliation-edge precedent), so the tax shows in
  the *seller's* P&L (a `tax` line) and the treasury merely accumulates (no
  appropriation path). (The general payer-side remittance-split seam from
  Phase 3 stays for tips/fees.) The **general store's consignment sale**
  (see [retail.md](./retail.md)) is a second split consumer: one `settle`
  routes the ask's remainder to the consignor's primary account (a
  `consignment` `PnlCategory` leg) while the store keeps the commission as
  its taxable revenue (`remitDemoTax` on the commission only) — the store
  fronts no coin, conservation holds.
- **The P&L** — `BankingApi.profitAndLoss(account)`: a derive-on-read
  categorized read (per-category signed net + running balance) — the
  deficit-as-target instrument, red by design. `house pnl` (operator-gated)
  reads it; `reserve mint <amount>` (the CB faucet) mints
  `subsidy` into the venue account to cover the red — a logged, visible,
  accountable faucet. **`reserve` is now Governor-gated** (`requiresGovernor`
  / holding the `central-bank-governor` office — the founder by default, or
  a handed-off holder), no longer `requiresWizard`: minting money is a
  monetary-authority act, not a code-trust one. This realizes the "governance
  of the central bank" `CentralBank.ts` left deferred — see
  [governance.md](./governance.md). (`house` stays operator-gated.)

## Reporting consumers + the bar loop (Phase 5)

Queryability is a property of the architecture, not a feature: the typed
append-only ledger + conservation + only-the-CB-mints means the two and only
two consumers are derive-on-read with no backfill:

- **The P&L** (Phase 4) — `profitAndLoss`.
- **Money supply + reconciliation** — all **per currency**; reports partition
  by currency and **never sum across** (with no rate, a combined total is a
  number nobody can justify).
  - `moneySupply(currency)` — Σ mints − Σ drains, O(1) off `SupplyAggregate`.
  - `reconcile(currency)` — the **circulating**-value audit, **sync**:
    `supply === Σ balances + Σ circulating coin`. It walks the in-memory
    index, so it sees live instances only, and it excludes vault float. Kept
    sync for the paths that cannot await, and now honest about what it omits.
  - `fullReconcile(currency)` — ⭐ **the follow-the-money instrument**, async.
    The identity that actually holds is
    `supply === Σ balances + circulating + snapshotCoin`.

    ⚠⚠ **Two accounting subtleties, both found by driving, both easy to get
    backwards:**

    1. **Vault float is REPORTED but NOT ADDED.** `seedFloat` mints coin into
       the till *and* credits the branch's own operating account 1:1 against
       it, so the float is **already on the ledger** as that balance. Adding
       `vaultCoin` to the identity double-counts it by exactly the float —
       the original `reconcile`'s `continue` on vault cash was **load-bearing
       accounting, not an oversight.**
    2. **`snapshotCoin` counts only NON-RESIDENT holders.** A snapshot is a
       *copy* of state that may also be live; if the holder's avatar is still
       materialized, `liveCoinOf` already counted those coins and counting the
       record too doubles them.

    The snapshot term is the one the audit genuinely was missing — it is
    where a logged-out player's cash lives, and nothing on the ledger
    corresponds to it.
  - `reserve supply` (governor-gated) renders one block per registered
    currency from the **complete** read.
- **The account statement** (player-facing) — `bank statement [count]`, a
  `BankController` subcommand beside `balance`: a derive-on-read scan of the
  caller's own ledger at the branch they're standing in (`entriesFor(accountId)`
  — the substrate reader `profitAndLoss` also rides), rendered newest-first
  with a per-line running balance accumulated over the *full* history, the
  most recent `count` rows shown (default 20, capped 100). No new substrate —
  the append-only ledger makes it a pure read; the P&L's ledger-scan sibling
  turned toward the account holder rather than the venue owner.

**Cash genesis** — `issueCash(into, amount, category?)` is the CB physical-cash
faucet: a `mint` issuance → cash bridge (supply grows, no account touched) plus
a `Coin` stack cloned into the world. The only ways coin enters circulation are
`issueCash` and a `withdraw`; both keep `reconcile` balanced. The optional P&L
`category` tags the mint (e.g. the char-gen **onboarding coin** → `onboarding`).

**Transit-fare consumer (the TPA).** The fast-travel fare is the **first wired
consumer of `settle`'s remittance-split seam** (see
[fasttravel.md](./fasttravel.md) § The transit-fare economy): a paid ride's
`Charge` credits the city-budget account `fee − networkFee` and splits the
network fee to the `fasttravel.tpaAccount`, conserved, refused on short funds.
It also drives the **on-ledger cash bridge**: a cash `settle` with no
`payeeContainer` (a global TPA account has no local till) **banks the coin
instantly** — the coin is consumed (`drainCoins`) and the equal value posted
`cash bridge → payee (+ splits)` as a `deposit`, supply-neutral — so a cash
fare splits identically to a credential fare (D12).

**The bar loop** — open an account → deposit cash → order a drink → pay (or
tab + settle) → the bar's ledger accrues sales/wages → it runs red → the CB
mints subsidy to cover it → the P&L shows the deficit; `reconcile` holds
throughout. The drink purchase is wired live: `OrderController`, after
crafting, settles a **presented Charge** at the Menu's authored `priceFor`
(silent pay from the credential's active account) and the bar `remitDemoTax`;
an unpriced recipe is served free (backward-compatible). The bar's P&L
account is **lazily ensured** (`ensureVenueAccount`, owner = the venue's
durable path) on first order/pnl/payroll — no boot seeder.

**Law compliance** — Law 1: no good carries a readable worth. ⚠ A coin's
`denomination` is a **number** (its face value in minor units) — read that
as the denomination's *structural key within its currency*, not as an
assertion of worth: the good does not price itself, the **currency**
validates and prices it (`Currency.faceValueOf`), and a
`(currency, denomination)` pair that does not resolve **throws** rather than
being worth what it says. Law
2: banking installs **no** scheduled recompute touching balances/coin (the
renown divergence), so nothing decays — an idle balance/stack is unchanged
over a game-clock advance.

## Module layout

`lib/banking/` (the new subsystem folder):
- `Currency.ts` — **the currency record + registry**: denominations (face
  value ⊕ per-coin mass ⊕ optional label), render vocabulary, issuer +
  governing office. ⚠ No rate, no cross-currency reference, permanently.
- `Money.ts` — the amount value-object (minor units + a required currency).
- `Coinage.ts` — the make-change **algebra** only (`dispense` / `planSpend`);
  it holds **no currency data**, reading the table off the record.
- `Account.ts` — the account-id vocabulary (sentinels, CB account, classify
  / mint helpers as statics).
- `Transaction.ts` — `BankTransaction`, the pure conservation rule
  (`assertConserving` / `supplyDelta`) + the `LedgerLeg` shape.
- `LedgerEntry.ts` — the append-only `bank_ledger` row (`Document`).
- `AccountBalance.ts` — the materialized account registry + balance
  (`bank_accounts`, warm cache; registry folded onto the row — decision 2).
- `SupplyAggregate.ts` — the supply headline, **one row per currency**
  (`bank_supply`, unique index on `currency`).

`obj/Coin.ts` — the physical cash object (`GlobbableMixin(Thing)`); a
concrete content object beside `Flask`/`AirTank` (memory: *obj vs lib Stuff
placement*).

`obj/CentralBank.ts` — the singleton mint/sink + world-presence anchor
(`PostRegistrationMixin(Idea)`; mint/drain logic surfaced through
`BankingApi`, developer-gated at the verb layer).

`api/banking.ts` — `BankingApi`, the thin gated forwarding shell.
`obj/api/BankingLogic.ts` — the `@Unshadowable` logic singleton at
`/obj/api/banking`, gated `FromModule('/api/banking#BankingApi')`; the
sealed `postTransaction` chokepoint lives here as a module-private fn.

Collections (`backend/PersistenceManager.ts`): `bank_ledger` (indexed
`fromAccount`/`toAccount`/`kind`/`at`), `bank_accounts` (unique `accountId`,
indexed `owner`/`bank`), `bank_supply` (**one row per currency**, unique
`currency`). All three carry a persisted `currency`.

⚠ **Boot ordering is load-bearing**: `BankingApi.boot()` runs **before**
`AccountBalance.warm()` / `SupplyAggregate.warm()` in `AppBootstrap`, because
`boot()` carries the idempotent currency backfill and **both warms throw on a
currency-less row**. Booting on an unmigrated database is therefore a fast,
harmless failure rather than a world running on money whose denomination
nobody knows — which is what makes the deploy order (**stop → migrate →
deploy → start**) enforced by the code instead of remembered from a runbook.
The `/obj/CentralBank` singleton is in the bootstrap manifest.

## Live-data migration (the currency build)

`packages/server/scripts/migrate-currency.ts` — four backfill targets,
because money lives in four places and only one is a collection banking owns:
the three `bank_*` collections; the well-known fixed account ids (`treasury`,
`central-bank`, suffixed **unconditionally** — a conditional would be exactly
the zorkmid branch the acceptance test forbids); the **`/obj/Coin` row in
`domain`** (⚠ the seeder is INSERT-ONLY, so unmigrated it keeps stamping every
*future* coin stale); and the nested coin blobs in `holder_snapshots` (the
durable coin population — vault float does not survive a restart, since
`BankCounter` composes no `PersistableMixin`).

The rehearsal is report-only → apply → report-only against a **restored copy**,
and the script **exits non-zero** unless every per-account balance, every
ledger row and every coin has identical value before and after. ⭐ The coin
term is the one that catches a silent revalue.

## Open-choice decisions log

The plan flagged 6 open implementation choices; settled as reached:

1. **Mint balancing** — chose the **latter** (supply tracked explicitly by
   `SupplyAggregate`; mint/drain are supply-changing single-leg postings,
   deposit/withdraw are cash-bridge postings, the rest zero-sum) over the
   former (a finite pre-seeded CB "unissued reserve" making every posting
   literally zero-sum). *Why:* the cash↔balance bridge is single-sided
   regardless, so the former's "every posting is zero-sum" elegance buys
   nothing once deposits exist; the latter is cleaner and makes
   deposit/withdraw first-class. (Phase 1.)
2. **Account registry home** — **folded** the registry fields (owner /
   bankPath / corpoKey / isPrimary / isActive) onto the `AccountBalance`
   row in `bank_accounts`, rather than a separate `Account` Document. The
   `RenownStanding` precedent (one collection carries both key and value);
   `Account.ts` is a pure id-vocabulary value-object, not a Document.
   (Phase 1.)
3. **Tab persistence** — chose the **mixin** (state on `TabMixin`,
   session-durable, the credential precedent) over a `Tab` Document. v1
   tabs don't need to survive a restart; the cross-restart `Tab` Document is
   the deferred option if required. (Phase 4.)
4. **Crafting price source for the presented Charge** — confirmed crafting's
   `order`/`serve` exposes **no** price stance (it crafts the drink free), so
   per the requirements' "authored flat stances" the bar's drink prices will
   be an **authored price field on the `Menu`**, built into a presented
   `Charge` and settled via `settle`. The settle primitive + Charge ship in
   Phase 3; the Menu price field + the order→charge wiring land with the bar
   loop in Phase 5 (the integration point). (Phase 3.)
5. **Verb category / shape** — one `banking/` category, and (post-review)
   **dispatch-on-subcommand parents, not a verb-per-action** (the
   `chat`/`alias` precedent — the framework supports far more than the
   old-MUD flat-verb style): `bank`
   (open/deposit/withdraw/transfer/balance/statement),
   `wallet` (use/freeze), `tab` (settle/skip), plus the flat `pay`; the
   operator surface splits **`reserve`** (mint/supply, central bank) vs
   **`house`** (pnl/payroll, venue owner): `house` is `requiresWizard` via
   `AuthorMixin`; **`reserve` is `requiresGovernor`** (the
   `central-bank-governor` office — see [governance.md](./governance.md)),
   the monetary-authority axis rather than code-trust. 13 flat verbs → 6, collapsing most verb collisions
   (no more banking `open` shadowing the boundary `open`). A pure view-layer
   regroup — the Api/Logic substrate is untouched. (Phase 2 surface,
   restructured post-MR review.)
6. **Branch/teller homing** — city content under `seeds/domain/eternal/`
   (resolved by the plan). Sub-choice settled: **`BankMixin` on a
   teller-counter `Thing` fixture**, NOT on the branch Location — a
   Location's own `commandContributions` don't reach its occupants, so the
   affordance must come from a fixture in the room (the `Menu` precedent).
   (Phase 2.)

The currency build added three more:

7. **Registry over data-Idea + catalogue** for `Currency` — `faceValueOf` is
   on the hot path of `Coin.getMass()` and the scope-walk keyword getter and
   now *throws*, so an async-warmed catalogue would resolve nothing in every
   test that skips the clone pipeline; and the test-only fixture currency
   must never reach a live collection, which a registry gives by
   construction (records are code, not rows). The door to authored
   currencies stays one seam wide.
8. **`Money.of`'s currency is required, not defaulted** — a default lets a
   site silently assume the wrong currency, caught only at runtime; required,
   the compiler enumerates every site. 174 call sites, deliberately.
9. **Currency joins the account-resolution key** — accounts are
   single-currency, so `(bank, currency)` is the identity. It keeps the
   scalar `balance` and its warmed cache untouched, where a multi-currency
   wallet would have rewritten every read site.
10. **The `setQuantity` gate lives on `Coin`, not on `GlobbableMixin`** — a
   glob is not necessarily money, and gating the mixin gates every pile of
   ore in the world. `Coin` is the value-bearing glob (and scrip will be a
   `Coin`), so it inherits. A general *value-bearing* marker is the
   money-integrity slate's own cycle.
11. **The clone pipeline is on the gate's allow-list.** A `Hydrator` applies
   a template's (or a snapshot's) `quantity` through the two-phase
   `set<Field>` dispatch, so hydration is how a coin stack comes into
   existence at all — for a fresh clone *and* for a logged-out player's cash
   restoring from `holder_snapshots`. Reconstituting a stack that already
   exists is not minting. ⚠ The consequence, accepted knowingly: a template
   authored with `data.quantity: 1000000` clones into a fortune. That is a
   **content** surface (`class`/`hydratorClass` are wizard-gated), and it is
   enumerated in the money-integrity slate.
12. ⚠⚠ **Vault float is reported, never added, to the conservation
   identity** — it is already on the ledger as the branch's own balance
   (`seedFloat` mints the coin *and* credits the account 1:1). The original
   `reconcile`'s skip of vault cash was correct accounting that the currency
   build briefly "fixed" into a double-count. **Before adding any term to a
   conservation identity, ask what already represents it.** The same hazard
   applies to `snapshotCoin`, which is why it counts only *non-resident*
   holders.

## Deferred seams

Lending / fractional reserve / interest / bank-failure + deposit insurance;
the governed reserve (legislative fiscal cycle); the employment
relationship (banking ships wage *payment* only); live/governed taxation;
player-run banks; the corpo faction-approval consequence of an account's
affiliation; the grey market; a full reporting/analytics surface (v1 ships
only the P&L + supply/reconcile consumers).

---

## The Attendant + Goodkin cycle (Terms · quota · royalty · coinage · re-home)

Additions from the Attendant-substrate cycle. See
[attendant.md](./attendant.md) for the storefront-attention substrate the
Goodkin bank runs.

- **The tab is gone.** The soft-credit `TabMixin` (skippable, redundant with
  pay-per-drink) was removed — **zero credit anywhere** until credit is designed
  for real.
- **Coinage — 1 / 5 / 25 (`lib/banking/Coinage.ts`).** The civic credit mints in
  three denominations (`credit`/`crown`/`sovereign`); a coin's **per-coin mass is
  derived from its denomination** (`Coin.getMass` → `Coinage.perCoinMass`,
  currency-intrinsic like face value, robust across clone/split/merge). Two
  make-change routines: `dispense` (largest-first, for mint/withdraw) and
  `planSpend` (bounded exact-selection, for cash payment). `issueCash` /
  `moveCoins` / `drainCoins` / `cashOnHand` / `withdraw` and the cash settlement
  path are value/denomination-aware; withdrawal + cash payment **refuse
  gracefully before any ledger leg** when exact change is impossible
  (till-can't-make-change / **exact-cash-or-card**). Wealth is now physical — a
  fortune is too heavy to carry (the mass cap), so it must go on the weightless
  ledger.
- **Terms (`lib/banking/Terms.ts`).** A per-bank fee/minimum schedule authored on
  `BankMixin` (`data.terms`), read at each verb. Every fee is a **conserved leg**
  (customer → the branch operating account), with a corpo **royalty** split off
  the top to the corpo treasury (`ensureCorpoTreasury`, keyed on `corpoKey`) — so
  corpo income begins from the first fee. The core custodial loop (open, deposit,
  withdraw your own money) is free everywhere; fees live only on
  movement/convenience. Goodkin's schedule is near-fee-free (the one live fee is a
  light cross-corpo wire).
- **The withdrawal quota (common-pool till guard).** A per-account daily
  cash-withdrawal cap, **derive-on-read** over the ledger's `withdraw` legs since
  the game-day boundary (no counter, no scheduler — Law-2 clean), **never
  collective** (a bank run is a feature), **scaling with standing** (Circle
  membership → the raised cap; set by `enrollCircle`). Circle membership is an
  attribute of the **member**, not the account — a `<corpoKey>.circle` saved
  boolean prop on the player (the general `PropertiedMixin` every `Creature`
  carries), read off the withdrawing principal, never a field on
  `AccountBalance`.
  The sibling of the Attendant lease (exclusive) — the common-pool guard.
- **Till security.** `BankMixin.canRemoveContainable` vetoes loose removal of
  vault coin; the banking verbs open a disbursement window
  (`_beginDisbursing`/`_endDisbursing`, gated to the banking module) around the
  one legitimate withdrawal, so "loot the drawer" is impossible.
- **The opening float.** A branch's founding capital: mint coin into the till
  AND credit the branch's own operating account against it (backed 1:1 →
  conservation holds). Seeded **lazily inside `openAccount`** on the first
  customer at the branch (the counter is guaranteed live then, which a boot-time
  seed can't guarantee), idempotent — a module-internal op, not a public verb.
  Lets early ledger-credit withdrawals work before deposits accumulate.
  `banking.openingFloat` AppSetting.
- **Goodkin re-homed.** From the placeholder `/domain/eternal/university-avenue/bank`
  into the Terminus **Counting-Houses** (`domain/terminus/counting-houses/`): a
  Locality + zone + a public avenue block (the four rival frontages as prose) +
  the banking hall (counter + Wenna) + the Circle parlor (Halloran). A complete
  **`Business`** (teller + officer positions, roster = hours, one P&L). The
  **enrollment** is a `tree-dialogue` on Halloran (verb-guided: the officer walks
  you through the real `bank open`/`bank deposit`/`pay` verbs) with a
  **`bank-circle`** dialogue effect that enrolls you into the Circle — registered
  from `lib/banking/BankDialogueEffect.ts` into the generic
  `DialogueEffectRegistry` (banking → the dialogue substrate, so the substrate
  never imports banking; see [npc-dialogue.md](./npc-dialogue.md)). The old
  university-avenue seeds are retired (a live-DB reseed is delete-and-restart;
  fresh DBs correct automatically).

## The leg-kind vocabulary (work-contracts build)

The `LedgerKind` vocabulary is now **closed and runtime-validated**:
`LEDGER_KINDS` (`lib/banking/LedgerEntry.ts`) is the source array the
type derives from, and `assertLegKind`'s switch carries an
exhaustiveness backstop — a kind the switch doesn't name **throws**
instead of falling through unchecked (the audit's one former hole; a
new kind is a compile error until its counterparty rule exists). The
"no untyped legs" acceptance is a pinned test.

The twelve kinds and their counterparty rules are the table in
[Conservation](#conservation) above. The four added by this build:

- **`escrow-hold` / `escrow-release` / `escrow-revert`** — the contract
  stake family (issuer → escrow, escrow → contractor, escrow → issuer).
  Escrow is an **agent's account at a commercial bank**: the contract
  system holds the stake in a **per-contract REAL account**
  (`Account.escrowAccountFor(id)` → `escrow:contract:<id>`, owner =
  `contract:<id>`, custodied at the default bank). Deliberately NOT a
  sentinel — a sentinel has no balance row and would take held funds
  *out of* the reconcile audit while in flight; a real row keeps
  `reconcile` green with **zero** changes to the audit. And deliberately
  not pooled: per-contract, an over-release **throws** on insufficient
  balance instead of silently robbing another contract's stake, and the
  per-contract conservation read (`escrowBalanceOf`) is the legibility
  surface ("the stakes are real because the money is locked and
  visible"). `escrowClose` asserts zero and deletes the row at every
  contract terminal — live escrow rows scale with *open* contracts; the
  ledger legs are the permanent record. See
  [contract.md](./contract.md).
- **`draw`** — the proprietor's take-home (business → proprietor's
  primary), `BankingApi.payDraw`. **Solvency-checked** — refuses when
  the business balance is short — where `payWage` deliberately pays red
  (the deficit model): a worker's wage is owed regardless; an owner
  pocketing from an insolvent business is exactly the wedge the
  distinct kind exists to expose. Surfaced as the wallet-afforded
  `draw <amount>` verb (business resolved *from* the acting proprietor
  via `EmploymentApi.businessOfProprietor` — no parameter to spoof).

**The two-layer discipline (kind vs category).** `kind` is the
**conservation/counterparty class** `assertLegKind` enforces;
`category` (`PnlCategory`) is the **economic line** reports partition
on. The rule: **a standalone movement gets its own kind; a rider split
inside a multi-leg transaction gets its own category** — forced by
`postTransaction`'s one-kind-per-transaction shape. The consignment
sale split (kind `payment`, category `consignment`) is the shipped
precedent; the share-of-flow compensation split (kind `payment`,
category `commission`) and piece-rate pay (kind `wage`, category
`piecework` — labor income; the wage-vs-draw tax wedge stays a *kind*
distinction) follow it. New categories this build: `escrow`, `draw`,
`commission`, `piecework`.

**This vocabulary is the future tax-policy hook**: a governance rate
table over kinds/categories (wage vs draw vs commission taxed
differently) reads these rows with no rework — the reason every
economically distinct movement must carry a distinct, named leg.

`postTransaction` now returns the minted `txId` (empty offline), so a
consumer's own event chain can reference its money legs — the contract
events' `txId` link.

## Every account names a real custodian

No account is held *nowhere*. Before this build, venue/city/worker
accounts were self-custodied (`bankPath` = the owner's own non-bank
path) and the sales-tax `treasury` row had an empty one — a recorded
owner with **no accountable custodian institution** (no Terms, no cash
ops, nobody answerable if the books are disputed). The rule, aligned
with the constitutional line:

- **Only the state banks at the CB** (`Account.CENTRAL_BANK_PATH`). A
  CB account belongs solely to an organ of the polity; the sole current
  occupant is the `treasury` (the legislature's fisc — the account the
  future appropriation build draws from).
- **Everything else is private and banks at a commercial bank** — the
  `banking.defaultCustodianBankPath` AppSetting, seeded to the Goodkin
  branch (v1's one real commercial bank; diegetically the bank that
  wants the newcomer, apt for the worker wage-fallback). City
  governance inside the diegesis is *content, not the state* — Terminus
  is group-owned land, so the city budget is a private account like any
  other. Corpo treasuries already bank at their corpo's own branch.
- **Escrow follows the real model** — the contract system's
  per-contract accounts are custodied at the default commercial bank (a
  future seam: escrow custody as a competitive bank product on Terms).

**WHICH custodian is a relationship, never a code default.** The
custodian of each account class is derived from who the account is for:

- **A business banks where its authored `banksAt` says** — an
  **institution key** (`banksAt: goodkin`) on the Business seed (next to
  `proprietorPath`/`positions`), resolved through the ONE seam
  `EmploymentApi.operatingAccountOf(business)`. A business with no
  `banksAt` cannot open an operating account (an authoring error,
  refused loudly — the no-operator-to-collect-the-fare precedent). This
  is what makes the custodian choice *matter* when the other corpo
  banks land: your bank's Terms price your fees, and every fee routes a
  royalty to your bank's corpo. All seven former call-site defaults
  (bar/store income, the TPA fare operators, wage + piecework payers,
  the draw, `job post --business`) route through this seam. The six
  shipped Business seeds author `banksAt: goodkin` (one edit each to
  defect); the counting-houses Business banks at Goodkin — itself (the
  legitimate self-custody: the custodian IS a bank).
- **A worker's first account opens at the payer's bank** — the
  employer's `banksAt` (`ensurePayableWorker`): an **NPC** with no
  account is set up where its money first comes from. A **player is
  never silently signed up for a bank**: no primary account → the wage
  is skipped with a warning / the piecework settle refuses / the
  flow-split doesn't fire — they open their own at a branch.
- **Escrow is custodied at the ISSUER'S bank** — the per-contract
  account opens wherever the funding account is custodied (*your* bank
  holds *your* stake), pre-positioning escrow-as-a-bank-product on
  Terms. The default is only the last resort for a legacy funding row
  with no recorded custodian.
- **The Teleport Authority banks as a Business** — see the TPA note in
  [fasttravel.md](./fasttravel.md): a minimal TPA Business
  (`fasttravel.tpaBusinessPath`) owns the network-fee income at its
  authored `banksAt`; the legacy raw `tpa` accumulator row is re-owned
  to it at boot.

Mechanically: `ensureVenueAccount` **refuses** a custodian that names
no real institution (accepted: `central-bank`
(`Account.CENTRAL_BANK_INSTITUTION`), the default-custodian setting, or
an institution with a live `BankMixin` branch), and an idempotent
**boot restamp** (`BankingApi.boot`) migrates legacy rows — the
pre-institution `bankPath` branch paths map to institution keys (a live
branch's path → its `getBank()`, the CB path → `central-bank`, else the
default), the `treasury` row → the CB, the bare `tpa` row → the TPA
Business, anything still custodied nowhere → the default — a
**cache-field fill, never a money movement** (balances and conservation
untouched). `banking.defaultCustodianBank` (`goodkin`) is therefore
**the restamp's last resort, not a default to build on** — nothing else
should pass it. Behavior-preserving: these accounts move money by
transfer only (never the till), and liquidity≠solvency is already the
shipped branch-book property, so Goodkin's cash physics are unchanged.

> **History (work-contracts build, `3969a34e..d8389518`).** This build
> added the closed `LEDGER_KINDS` vocabulary + the escrow family +
> `draw` (the leg-kind sections above), then two review rounds reshaped
> custody: **custody is a relationship, never a default** (`banksAt` /
> payer-derived / escrow-at-the-issuer's-bank; the default became the
> restamp's last resort, read from `banking.defaultCustodianBank` with
> NO code fallback), and **the bank became an institution** — accounts
> re-keyed from `{owner, bankPath}` (a branch counter's templatePath) to
> `{owner, bank}` (an institution key; `BankMixin.getBank() = authored
> ?? corpoKey`). `AccountBalance.bankPath` survives only as a legacy
> hydration carrier the boot restamp migrates and clears — remove with
> the terminus-banking build.

## History

- **The currency build** (2026-08-05, `c1b9b711…52f65257`) — the money
  substrate generalized from one *implicit* currency to N, shipped with
  exactly one. `credit` → **zorkmid** (the word went to the deferred lending
  subsystem); denominations became **structural** `(currency, faceValue)`;
  `currency` threaded through all three `bank_*` collections; conservation
  became per-currency with a **permanent** no-crossing rule; the cash-side
  `setQuantity` hole was gated; and `reconcile` split into a sync
  circulating read and the complete `fullReconcile`.

  ⚠ Three defects in that build were found **only by driving the client**,
  not by the suite — enroll was broken by the new gate (the tests call
  `issueCash` from a root execution context and stub `clone`, so they never
  exercised the real caller or the hydration path), a coin stack rendered
  as *"4 a 5-zorkmid pieces"*, and the new vault term double-counted the
  branch float. The lesson is the one the project already holds: **green
  tests are not a working flow.**
