# Chattel — per-instance ownership of movable goods

The **possession core** for movable goods (property slate Phase 0b, the
chattel half): who owns a particular Stuff instance — a torch, a lantern,
a consigned good — stored unspoofably, surviving the item moving between
containers and logging out and back. It is the **twin of the parcel
title** ([parcel.md](./parcel.md)), one cardinality down: parcels title an
ownable *extent* (a zone path); chattel titles an ownable *instance*.

## The governing invariant (inherited from parcels)

Ownership is a row in a **separate, gated registry**, stored **apart from
the editable good it gates** — never a field on the item. Same reason as
parcels: a title must not be spoofable by editing the thing it describes.
The item carries only its **identity** (a durable id); its **owner** is a
registry fact, resolved through a facade Api, with the actor derived from
execution context (never a caller-supplied param).

## `ownerOf(item) = stamp ?? parcel-extent ?? authorOf`

A per-instance **owner-stamp** overrides. Absent a stamp, a good whose
template path falls under a **parcel's extent** resolves to that parcel's
owner — so a landlord owns the fixtures in a unit they let. Absent both,
ownership defaults to the item's **author** (via the shipped
`ProvenanceApi.authorOf(templatePath)` — [provenance.md](./provenance.md)).
So an unstamped content good outside any extent resolves cleanly to its
author with **no world-wide restamp**, exactly as it did before the middle
rung existed, and only goods that have actually changed hands carry a
stamp. `null` for a good that is none of the three (engine litter). A
three-rung total chain — the parcel `ownerOf`'s (title → self-home →
state) analog.

The parcel rung keys on the **template path, not the location**, which is
what makes displacement recoverable: a fixture carried out of a unit stays
titled to the parcel, so it is theft (custody without title), never a
transfer.

> ⚠ It is built on `ParcelApi.coveringParcelOf`, **not**
> `ParcelApi.ownerOf` — the latter is *total* (it falls back to the state)
> and would make the author rung unreachable, silently retitling every
> authored good in the world.

`ChattelOwner` mirrors `ParcelOwner` so a group-held parcel is
expressible. **Read-side only**: only the `player` arm is ever stamped, so
no stored row gains a group owner.

## Where the owner keeps it — `place`

A stamped good also carries a **`place`** (a room identity, `'inventory'`,
or `'storage'`) and persists **owner-side** rather than with whatever room
it is standing in. That is the [furnishing](./furnishing.md) subsystem —
`setPlace` / `followCustody` / `placedIn` / `evictToStorage` / `isStamped` /
`isOwnerPersisted` (stamped to a player — the persistence skip rule's predicate)
on this same facade, the `place` column on the `chattel` row as a by-room
index, and the estate slice that carries the goods themselves.

## ⭐ What identity means for a vessel — the row is provenance, the instance is the thing

⚠ **Added 2026-08-29** (the aluminium-can review; slate
`slates/builds/aluminium-can-slate.md`). A question worth settling
before the returns/deposit work: *drink the cola — what are you left
with, and is it the same object?*

**It is the same object.** The durable identity of a movable good is its
**instance** — the `_chattelId` below — not the template row it was
minted from. For a vessel the row says **how it was produced** (`a can,
filled with cola, 0.33 L`); it is provenance, not a standing claim about
what the thing currently contains. Drinking changes the contents, not
the identity: same instance, same chattel id, unbroken chain of title.

The alternative — a filled can and an empty can as two *stored*
identities, transmuting on drain — was weighed and rejected. It is the
more intuitive shape for shopping, and the engine already gives that
feel by **derivation**: an emptied vessel counts under its vessel kind
(`vessel:can`, [bulk.md](./bulk.md)), reads as *"The can is empty."*,
and is claimable by a fill of that kind. Stored transmutation buys
nothing extra and costs a great deal — re-stamping `templatePath` is the
identity anti-pattern, and destroy-and-create severs this chain of
title, orphans any consignment listing, and invalidates live refs, on a
routine action.

⭐⭐ **The decisive case is the deposit.** A deposit exists precisely
because a vessel has value *independent of what is in it*. If drinking
destroyed the product and created a new empty, there would be no answer
to *what did you pay the deposit on* — the object paid for is gone and
the object returned was never sold. Because the instance persists, the
deposit is an ordinary escrow leg against a good whose title never
broke: you paid a deposit on **this can**, and **this can** is what you
bring back.

## The durable per-instance id (the crux)

Chattel keys on a **durable per-instance id**, minted server-side on first
stamp, that survives the persistence round-trip. `ChattelMixin`
(`lib/chattel/Chattel.ts`) contributes:

- **`_chattelId`** — a declared **persistent field** (empty until minted).
  Because it is in `fieldMeta`'s persistent entries, the drift-guarded capture/restore
  ([persistence.md](./persistence.md)) re-applies it: the id rides the
  Avatar-inventory (or consignment-shelf) snapshot, and the registry row
  is independently durable in Mongo. A stamped good, logged out and back,
  resolves to the same owner.
- **`getChattelId()`** + a **gated `_setChattelId(id)`** (chattel-logic-only —
  `FromTemplate('/platform/idea/api/chattel')`, the minting authority named directly,
  `@Final @Unshadowable`) — so the identity is server-minted, never
  forged by an author or player. (The persistence hydrator bracket-assigns
  `_chattelId` on restore — it looks for `set_chattelId`, not
  `_setChattelId`, so the gate protects only *programmatic* writes.)
- **`onDestruct` GC** — when a stamped good is genuinely destroyed
  (consumed, explicitly destructed, or culled loose in a transient room),
  it releases its registry row (a terminal `released` event). Safe to
  release unconditionally because (1) ids are UUIDs, **never reused**, so
  a leaked/stale row can never be inherited by a future clone, and (2)
  player-held goods **evacuate** (move to the room) rather than destruct
  when their host is evicted — so `onDestruct` never fires on an inventory
  item whose id is riding a snapshot, and its durable row survives the
  relog.

Composed at the **`Thing`** tier (the movable-good tier), so every
portable inanimate object gets per-instance identity for free
(pets/apartments/ranching/retail all want it).

### Discrete goods only

The stamp is coherent only on a **discrete** instance. A `Globbable`
fungible stack is structurally incompatible — a split of a stack of five
has no answer for which unit keeps the id, and a merge equates identities.
So chattel is **discrete-goods only**: `stampChattel`/`transferChattel`/
`ownerOf` **refuse a glob** (a clear no-op, not a silent mint), a glob's
`_chattelId` stays empty, and fungible stacks are **owned-by-possession**
(whoever holds them). Coins are the money substrate, conserved by the
banking ledger, not the ownership registry.

## The trio (mirrors the parcel trio)

- **`ChattelApi`** (`api/chattel.ts`) — the thin gated facade:
  `ownerOf(item)`, `stamp(item, owner)`, `transfer(item, newOwner)`,
  `release(chattelId)`. Resolves the logic singleton via
  `StuffApi.singletonSync` + `HotReloadApi`. Actor context-derived.
- **`ChattelLogic`** (`platform/idea/api/ChattelLogic.ts`, `/platform/idea/api/chattel`,
  `extends ApiLogic`) — the `ownerOf = stamp ?? parcel ?? authorOf` chain, the
  glob-refusal invariant, and the **pure degrade** when no registry is
  live (the author fallback still resolves; mutators no-op). Gated
  `FromModule('/api/chattel#ChattelApi')`.
- **`ChattelRegistry`** (`obj/ChattelRegistry.ts`, `/platform/idea/ChattelRegistry`)
  — the singleton index (`chattelId → ChattelOwner` Map over the `chattel`
  collection) + the mint-a-fresh-id seam; the **sole writer** of `chattel`
  / `chattel_events`. Every method gated
  `AnyOf(FromModule('/api/chattel#ChattelApi'), FromTemplate('/platform/idea/api/chattel'))`.
  Warmed at boot via a `bootstrapManifest` entry (`postRegister` rebuilds
  the index from the collection).

## Storage

- **`chattel`** (`ChattelRecord`, `lib/chattel/ChattelRecord.ts`) — the
  rebuildable current-state row: `{ chattelId, owner: ChattelOwner, titledAt }`.
- **`chattel_events`** (`ChattelEvent`, `lib/chattel/ChattelEvent.ts`) —
  the append-only chain-of-title: `mint` | `transfer` | `released`, one
  row per act, `from`/`to`/`actor`/`at`. A transfer never overwrites
  history; prior owners stay recoverable. The terminal `released` event
  on GC is the **deliberate divergence** from parcel `retire` (a silent
  delete) — a destroyed good's chain-of-title should terminate legibly.

`ChattelOwner` is a typed principal (`{ kind: 'player'; templatePath }` in
v1; the union leaves room for a group/corpo owner without a migration).

## First consumer: the general store

Chattel is proven independent of the shop (stamp/transfer/ownerOf, author
fallback, survival across a container move + a persistence round-trip,
glob refusal, GC), then consumed by the store's **buy-that-stamps** and
its **custody-vs-ownership consignment** — see [retail.md](./retail.md).
Consignment is the load-bearing exercise: `consign` moves *custody* to the
shop while the owner-stamp stays put, and the sale transfers *both*.

## Deferred

The property compute/economy (cost-owner, the `Charge`-debtor gap,
allowance — property slate Phase 1); a general player-facing
`give`/trade/dispute surface (the `transfer` primitive makes it a thin
later add); group/corpo owners; fungible-goods ownership.

## Looting a corpse (2026-07-31)

The mortality build deliberately adds **nothing** here.

A dead body's gear moves onto the corpse, and taking it is ordinary
containment. The record of who took what is the **chain of title this
subsystem already keeps** — which is why looting mints no
`accountability_events` row: the ledger would be duplicating a record that
already exists in the right place, and inventing a kind for theft would
force a "what is theft" ruling the courts work should make.

The corpse itself carries **no chattel row**. `ChattelMixin` composes at
the `Thing` tier and a body is a `Creature`; custody of a *body* is a
different concept, and it belongs to the deferred coroner economy. See
[mortality.md](./mortality.md).
