# Credential

The unified credential substrate: **one holder, many credentials-as-data.**
Where the engine once grew a new mixin (plus an aether-hosted update and a
physical card) for every credential concept — `PaymentCredentialMixin`,
`TravelCredentialMixin` — a new credential **kind** is now a new *record*
held in one `CredentialWalletMixin`. The wallet is a dumb store; the records
carry the kind-specific authorization state; the verbs a credential lights up
live on the **holder**, not the record. Three credentials was the signal to
stop building mixins and build the holder.

Lives in `lib/credential/`.

## The two jobs a credential does

A credential conflates two separable jobs. Keeping them apart is what makes
"credentials as data" work:

- **Affordance** — holding the thing makes a verb *exist for you*. The old
  `PaymentCredentialMixin` carried `commandContributions` that pushed
  `pay`/`wallet` onto whoever held it. **Affordance rides the holder** (the
  wallet app / the card), via its static `commandContributions`.
- **Authorization** — whether a *specific* use clears (account linked, not
  frozen, within cap; destination registered). **Authorization rides the
  record**, read by the smart consumer that already exists (`BankingApi`,
  the TPA fork).

The travel credential only ever did authorization — its verbs (`register`,
`teleport`) come from the terminal and the general verb set, not the
credential. The payment credential did both.

## The records — `lib/credential/Credential.ts`

`CredentialKind` is a vocabulary (`payment` | `travel` | `key`);
`CREDENTIAL_KINDS` is its validation array. **Adding a kind is adding data.**

`Credential` is an abstract value-object base (NOT a `Stuff` — a plain object
the wallet owns, so its surface is ordinary methods, no proxy/persistence
reflection). It knows its `kind` and serializes itself to a plain
`SerializedCredential`. Two statics dispatch by kind: `Credential.mint(kind)`
(a fresh default record) and `Credential.fromData(row)` (rebuild from storage).

- **`PaymentCredential`** — the on-ledger record: linked-account set, active
  pointer, per-credential `spendCap`, `frozen` flag, and `authorize(amount)`
  (refuses over-cap or frozen — a security cap, never a fee). The behavior is
  lifted verbatim from the retired `PaymentCredentialMixin`.
- **`TravelCredential`** — the registered-node set plus
  `register`/`unregister`/`isRegistered`/`authorize`. Born with the
  **three-node `BORN_WITH_TRAVEL_NODES` floor** (the Terminus arrival node,
  the lounge, and the newbie-wilds crossroads — the University Avenue node is
  retired); `fromData` unions saved entries on top of the floor, never
  clearing it. **Presentation-vs-clearance down-payment:** the record
  satisfies the `teleport`/`register` *instrument* gate wherever it is held,
  but travel **clearance** is bound to the traveller's **identity** — read
  from and written to their aether-hosted wallet only, never a carried
  `TravelCard` (see [fasttravel.md](./fasttravel.md) § Identity-bound
  clearance). A card is a bearer *instrument/presentation*, never an
  authorization store — a narrow, non-ledger down-payment on the
  credential-wallet tail's "a credential is a presentation, not the source of
  truth" principle (the full issuer-authorization ledger stays deferred).
- **`KeyCredential`** — the **keychain**: a bag of **bearer** key entries
  (each a `{keyway, technology}` — an opaque lock-identity token + the lock
  technology it fits) plus a set of **master** technologies.
  `addKey`/`addMaster`/`removeKey`/`authorize(keyway, technology)` — it opens a
  lock iff it holds a matching entry, or a master for that technology (a
  brass key can't turn a keycard reader). The **same record** whether it lives
  in the implant keychain (the born-with wallet) or on a physical `Key` Thing
  (`lib/lock/Key`, a card carrying one entry). **Bearer, not credentialed:**
  possession *is* access — a lock is revoked by **re-keying** (a fresh keyway,
  so the old entry silently stops matching), not a per-key ledger. The lock/key
  substrate lives in `lib/lock/` (`Lock` value-object + `LockType` +
  `CredentialApi`); its first consumer is the dorm door — see
  [residence.md](./residence.md). **Durability:** the physical `Key` persists
  with its holder (carried inventory rides the spine); the implant keychain is
  session-durable in v1 (the physical key is the cross-restart form).

## The holder — `CredentialWalletMixin` (`lib/credential/CredentialWallet.ts`)

A keyed store of records (`Map<CredentialKind, Credential>`), base-agnostic so
it composes around both an incorporeal hosted update and a corporeal card.
Surface (`CredentialWallet` interface): `getCredential(kind)` /
`ensureCredential(kind)` (mints if absent) / `hasCredential(kind)` /
`getCredentials()`. The typed `getCredential<K>` returns the concrete record
for the kind (`CredentialByKind` map).

- **Persistence** rides the `credentials` accessor (the old `registered`
  idiom): the getter yields the storable `SerializedCredential[]`; the setter
  rebuilds records (`Credential.fromData`, so travel re-floors). Persistent
  field: `credentials`. Session-durable in v1.
- **Born-with seeding** — a concrete holder declares
  `static defaultCredentialKinds`; the `_credentials` field initializer mints
  an empty record per kind, so `new PaymentCard()` carries its record without
  a seed (the clone pipeline replaces them on hydrate). The mixin itself
  seeds nothing.

## The three holders

- **`CredentialWalletUpdate`** (`lib/credential/CredentialWalletUpdate.ts`,
  `CredentialWalletMixin(AetherHostedMixin(Idea))`) — the **born-with wallet
  app**, the single hosted update replacing the per-credential
  `PaymentImplantUpdate` + `TravelCredentialUpdate` twins. Injected once into
  an attunement host by `Avatar.installDefaultLoadout` (alongside comms +
  forums). `defaultCredentialKinds = ['payment', 'travel']`. Affords the
  payment verbs to its host via the `self` bucket; travel adds no holder-verbs.
- **`PaymentCard`** (`lib/banking/PaymentCard.ts`, `CredentialWalletMixin(Thing)`)
  — the carryable, losable, freezable 1:1 bearer instrument.
  `defaultCredentialKinds = ['payment']`; affords `pay`/`wallet` via the
  `inventory` bucket. Cloned per issue by `BankingApi.issueCard`.
- **`TravelCard`** (`domain/common/tpa/TravelCard.ts`, `CredentialWalletMixin(Thing)`)
  — a carryable TPA card: a bearer **instrument**, never a clearance store (its
  floored `travel` record satisfies the instrument gate; authorization is
  identity-bound). `defaultCredentialKinds = ['travel']`; affords no
  holder-verbs (the terminal affords `register`; the ticket clerk hands one out
  free via `procure card`).

The cards stay **thin per-kind subclasses** — they share storage, resolution,
and the holder mixin, and differ only in their static `commandContributions`.
This is deliberate: per-instance affordance (verbs varying by which record a
holder carries) exists today only for the `environment`/`peers` buckets
(`InstanceContributor`), not the `self`/`inventory` verbs a held thing gives
*you* — those are static-by-class. Keeping affordance kind-specific via two
trivial subclasses avoids touching the command hot path.

## Resolution — one scan, either base

Consumers no longer scan for a typed credential; the scan is an **MQL
`person` pool** (bearer semantics — the on-person seed: self + hosted
apps + slot occupants + carried; a card on the floor is never
"presented") filtered on `isCredentialWallet` + record-presence /
`!frozen` — they find the **holder** and read the record:

```ts
MqlApi.resolveMany('person', { commandGiver: actor, scope: 'person' })
  .stuff.find((s): s is Stuff & CredentialWallet =>
    MixinApi.isCredentialWallet(s) && !!s.getCredential('travel')) ?? null
```

`MixinApi.isCredentialWallet` (registry constant `Mixins.CredentialWallet`) +
a record-presence filter reproduces the old per-kind `isPaymentCredential` /
`isTravelCredential` scans exactly — including the **frozen-skip** (the
payment predicate filters `!record.isFrozen()`, so a reissued card is found
in place of a revoked one) and the **either-base** reach (a carried card via
the seed's inventory leg, the hosted wallet via its host-descent leg of the
three-base model — see [augmentation.md](./augmentation.md)); the seed's
on-person-first emission keeps the implant-first preference.

## Consumers

- **Banking** (`BankingLogic`) — `reachableCredential` resolves the holder,
  returns its `payment` record; `settle` / `openAccount` (auto-link) /
  `activeCredential` (context-derive the actor's routing credential) / `issueCard`
  read/mint it. `issueCard` returns the card holder (`Stuff & CredentialWallet`).
  Mutating a resolved credential (`setActiveAccount` / `setFrozen`) is the
  `PaymentCredential` value-object's own behavior — the wallet verb holds the
  record and calls it directly (no gate to route through `BankingApi`). See
  [banking.md](./banking.md).
- **Fast travel** — the `teleport` TPA fork and `register`
  (`TeleportController` / `RegisterController`), the
  `requiresTravelCredential` validator, and `FastTravel.renderDepartures` all
  resolve the holder and read the `travel` record. See
  [fasttravel.md](./fasttravel.md).

## Persistence

Session-durable in v1: the wallet update is re-cloned each login from its
`defaultCredentialKinds`, so a payment credential's links re-establish via
`openAccount` on relog and the travel set resets to its born-with floor. No
new collection — the records live in the holder's `data.credentials`.
Cross-restart durability rides future persistence work (aug-state
colocation; inventory persist-back).

## Deferred (the slate's remaining surface)

The credential-wallet slate is
**not fully absorbed** — this build shipped the holder + migrated payment and
travel; these remain its design space:

- **Deputization** as a native tenant (the slate's first intended tenant,
  descoped from this build) — wired to `AccessApi` via an MQL group over a
  derive-on-read authorization ledger.
- **The issuer-authorization ledger** — the slate's "credential is a
  *presentation*, not the source of truth": validity derived from the
  issuer's append-only `authorize`/`revoke` ledger, the record being the
  boarding pass, not the clearance. v1 records ARE the source of truth.
- **A single `CredentialCard`** — the slate's "one card, polymorphic on its
  record." v1 keeps two thin per-kind subclasses (the affordance constraint
  above); collapsing to one class waits on per-instance `self`/`inventory`
  affordance.
- **A thin `CredentialApi`** (the slate's open dial 2) — not built; the
  holder's plain methods are consumed directly by banking / fast travel.

## Module layout

```
lib/credential/
├── Credential.ts            CredentialKind vocab + Credential base
│                            + PaymentCredential / TravelCredential records
├── CredentialWallet.ts      CredentialWalletMixin (the holder) + interface
├── CredentialWalletUpdate.ts  the born-with hosted wallet app
└── __tests__/Credential.test.ts, CredentialWallet.test.ts

lib/banking/PaymentCard.ts          CredentialWalletMixin(Thing), payment
domain/common/tpa/TravelCard.ts     CredentialWalletMixin(Thing), travel
```

Registry: `Mixins.CredentialWallet` (`lib/mixin.ts`),
`MixinApi.isCredentialWallet` (`api/mixin.ts`),
`TemplatePaths.credentialWalletUpdate` (`lib/paths.ts`).

## Cross-references

- [banking.md](./banking.md) — the `payment` credential consumer (settlement,
  the risk ladder, the `wallet` verb).
- [fasttravel.md](./fasttravel.md) — the `travel` credential consumer (the
  TPA fork, `register`, the scan-to-register unlock + born-with floor).
- [augmentation.md](./augmentation.md) — the three-base capability model and
  the aether hosting relation the wallet update rides.
- credential-wallet-slate.md —
  the design surface; the deferred portion above.
