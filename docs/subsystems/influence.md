# Influence: the three-stock standing substrate

How contribution and conduct become **governance standing**. Influence has
three stocks — **consumer** (play), **producer** (make), and **capital**
(fund) — three dimensions of every member, each earned and spent
independently. This doc covers the cross-stock substrate, the **producer**
faucet, and the **conviction** spend half. The two halves of the consumer
stock have their own docs: [participation.md](./participation.md) (the
quantity faucet + the `engagement × renown` projection) and
[renown.md](./renown.md) (the quality measure). The producer faucet's
authorship input lives in [provenance.md](./provenance.md).

This build ships the producer faucet (measured standing only — it feeds no
vote yet) and the conviction substrate (Api + tests, no verb). The ballot,
the chambers, and the capital faucet are deferred.

## `lib/standing/` — one home

The standing value objects share one directory, `lib/standing/`:
`Band`, `InfluenceStanding`, `ParticipationEvent`/`ParticipationStanding`
(consumer), `RenownEvent`/`RenownStanding` (renown), `ProducerEvent`/
`ProducerStanding` (producer), `Position`/`ConvictionTally` (conviction),
plus `AuthoringEvent` and `CreditRouting` (provenance). They are **one
system** — how contribution and conduct become standing. Named `standing`
(not `influence`) because the value objects already *are* `*Standing`, and
**renown is not subordinate to influence**: it has its own non-influence
readers (NPC reactions, disguise, social), so a neutral name is correct.
The `*Api` faces stay in `api/`; the `*Logic` singletons in `obj/api/`.

> **Durable identity (Phase 0, already merged).** The whole engagement
> cluster — renown, participation, producer — banks standing on the durable
> `templatePath` (`/obj/Avatar/<playerId>`), never the ephemeral `stuffId`
> (which is re-minted on re-clone). The id stays a live-resolution *handle*
> for scope; only the stored aggregation *key* is the templatePath. Renown
> and participation re-key together — the consumer projection multiplies
> them on one key.

## The three-stock output contract

Every stock emits the same shape, read uniformly by the future
chamber-weighting; only the `stock` tag and the faucet that produced
`scalar` differ:

- **`InfluenceStanding`** `{subject, stock, scalar, band}` — `stock` is
  `'consumer' | 'capital' | 'producer'`. `scalar` is the host-internal
  magnitude (reserved for the ballot); `band` is the player-facing tier.
- **`Band`** — a stock-agnostic qualitative tier (register D6: standing is
  shown as a band, never a grindable number). Cutoffs from AppSettings.
- **`InfluenceApi`** (`api/influence.ts`) — the thin cross-stock dispatcher,
  no faucet, no logic singleton. `standingOf(subject, stock)` delegates
  `'consumer'` → `ConsumerApi`, `'producer'` → `ProducerApi`; `'capital'`
  stays a reserved value returning a defined zero standing (the Twitch
  intake build), never a throw. `bandOf` reads the delegated band.

The symmetry lives at the standing/band layer; the **asymmetry stays at the
source** — capital is `concave($)`, not `engagement × quality`, so no shared
formula fits. Raw logs stay per-faucet.

## The producer stock (the make faucet)

An author earns producer standing from **engagement their released content
draws** — others spending time with what they built.

```
CommandDispatchedEvent ─► CreditRouting.resolve ─► producer_events ─► producer ─► InfluenceStanding
   (shared, receive-gated)   (author + released gate)  (append log)    (rebuildable)  ('producer')
```

- **`ProducerEvent`** (`producer_events`) — the append-only log, one row per
  attributed-engagement bucket. Fields: `author` (the credited author's
  durable `templatePath` — the **routing key**), `actor` (the engaging
  player's templatePath), `zonePath`, `weight` (default 1), `bucket`
  (`floor(realAt/bucketMs)`), `kind: 'engagement'`, `at`/`realAt`. Append is
  find-or-skip on **`{author, actor, bucket}`** — one player engaging one
  author's content credits that author **once per bucket** (anti-inflation).
- **`ProducerStanding`** (`producer`) — the rebuildable per-author
  aggregate (subject = author, `scope = '*'`): the recency-decayed,
  weight-scaled attributed-bucket count. Drop it, replay the log, get
  identical standings.
- **`ProducerApi`** (`api/producer.ts`) → **`ProducerLogic`**
  (`obj/api/ProducerLogic.ts`, `/obj/api/producer`): `boot` / `append` /
  `recompute` / `producerOf(authorId)` / `standingOf(authorId)` /
  `eventsFor`. Gated `FromModule('/api/producer#ProducerApi')`.
- **Formula is engagement-only** — the decayed, weighted attributed-bucket
  scalar, **no `× regard`** (an author earns from *draw*, not from being
  liked), so `ProducerLogic` reads no `RenownApi`. **Real-time** decay (the
  participation divergence — production standing measures *current* draw).
- The credit routing — *who* gets credited and the released-content gate —
  is `CreditRouting`, in [provenance.md](./provenance.md). The faucet
  (*how much*) is built once; v1 routes a single owner, the deferred team
  split returns a weighted vector behind the same `CreditShare[]` seam.

The per-Player → User rollup (the franchise-gate sum, the Sybil anchor) is
**not stored** — production doesn't dilute under a sum, so banking stays
per-Player; the rollup is a deferred franchise-gate read.

### The shared signal + the `restrictSubscribe` pair

Producer reuses the consumer's `CommandDispatchedEvent` (the recognized,
interactive dispatch from `CommandGiverMixin._emitInputEcho`) — no second
signal. The payload now carries two **optional** fields,
`locationTemplatePath` and `actorTemplatePath` (the giver's location + own
templatePath), read at the fire block; the consumer tap ignores them, so its
participation rows are byte-identical whether present or not.

Both `ConsumerLogic.installDispatchTap` and `ProducerLogic.installEngagementTap`
assert the **same full pair** `restrictSubscribe(KIND, ConsumerLogic,
ProducerLogic)`. This is load-bearing: `restrictSubscribe` clobbers the
policy with the latest call's list and its owner-guard refuses a subset, so
a single-class re-assert (after HMR, in any order) would silently evict the
other tap. Asserting the full pair from both makes every re-assert the full
set (a dedicated regression test guards it). The engagement tap routes credit
via `CreditRouting.resolve(location)` and credits each share's author
**unless `author === actor`** (the **A≠P** self-credit exclusion — engaging
your own content earns you nothing).

## Conviction (the spend half)

A holder commits standing toward a target (a bill / question); the **weight
that commitment carries is derived from how long it has been held unbroken**
(a linear ramp), never stored — so conviction rewards sustained commitment,
and a flip restarts the clock. No verb consumes it yet (the chambers are
population-deferred); this is the substrate + its tested clock seam.

- **`Position`** (`positions`) — one row per `{subject, stock, target}`;
  `{yea, nay, since, realSince}`. `subject` is the holder's durable
  `templatePath` (the same key `InfluenceApi.standingOf` reads). No stored
  weight is authoritative.
- **`ConvictionTally`** — the immutable per-house bill number.
- **`ConvictionApi`** (`api/conviction.ts`) → **`ConvictionLogic`**
  (`obj/api/ConvictionLogic.ts`, `/obj/api/conviction`): `hold` / `flip` /
  `drop` / `abstain` / `positionOf` / `tally` / `quorumWeight`. Every method
  takes an optional `now` (real-time MS) defaulting to the wall clock — the
  deterministic clock seam the tests drive. Gated
  `FromModule('/api/conviction#ConvictionApi')`.

The math:

- `conviction = clamp01((now − realSince) / buildPeriod)` — a linear ramp
  0→1 over one build period. A **new** position or a **net-direction flip**
  (re)starts `realSince`; a same-direction re-hold keeps building. `flip`
  swaps yea/nay and resets the clock; `drop` deletes the row.
- `tally(stock, target)` = `Σ standingOf(holder, stock).scalar × conviction
  × (yea − nay)` over the target's positions — the decision number.
  **Full weight / no pool** (each position spends the holder's *whole*
  standing scalar; `hold` never consults a holder's other targets) and
  **non-fungible** (partitioned by `stock` — the three houses tally
  independently; a consumer stake and a producer stake on one target are
  distinct rows).

### Present vs absent (quorum) — abstain

The **existence** of a `Position` row means the holder **cast a vote**
(counts toward quorum); absence means **not voting** (absent). An
**`abstain`** is a present, net-zero stake (`yea = nay = 0`): it counts for
quorum at the holder's **full standing** but contributes 0 to the decision —
so a founder with a supermajority can decline to take a side without starving
quorum. `quorumWeight(stock, target)` = `Σ standingOf(holder, stock).scalar`
over every present position (directional or abstain) — the participation
numerator a passage rule measures against the total possible. Quorum presence
is **conviction-independent** (you don't build conviction to show up);
conviction scales only the decision weight. The constitution (draft Art. IV)
sets the quorum-against-full-count rule this encodes.

## The `standing` self-view

`obj/command/social/StandingController.ts` — a zero-arg, self-only,
read-only self-view (afforded by `PersonaMixin`, sibling to `chronicle`).
Renders all three stocks as **bands** — **Play** (`consumer`, with its
participation + regard inputs described qualitatively), **Make** (`producer`),
**Fund** (`capital`, "not yet earnable") — keyed on the actor's durable
`templatePath`. Bands throughout, never raw scalars (D6).

## Tuning (AppSettings, no code defaults)

- `producer.bucketSeconds` / `producer.decayHalfLife` — the producer
  faucet's bucket width + real-time recency half-life (mirror participation).
- `conviction.buildPeriodSeconds` — the linear conviction ramp period.
- `influence.bandThresholds` — **reused**, stock-agnostic band cutoffs.

New `Collections`: `producer_events`, `producer`, `positions`. The
entrenched invariants (engagement-only producer formula, full-weight/no-pool,
non-fungible, derive-don't-track / rebuildable) are **code, never keys**.

## Deferred (named, not built)

The **ballot / chambers / 2-of-3 / delegation** (population-deferred); the
**capital** faucet (Twitch intake); the **producer team / co-authorship
split** (the weighted `CreditShare[]` vector + the richer team-sandbox +
explicit `release` action); the **second-order engagement** quality
enrichment (register D2); the per-Player → User franchise rollup; a
conviction verb. Producer feeds **no vote** this build — measured standing
only.


## The standing witness

After each append is persisted, the ledger calls
**`MqlSubscriptionApi.notifyDurableSubject(subject)`** — a direct method
call on the one consumer that cares.

Every standing here derives on read, which works fine for a verb (you
ask, it computes) and not at all for a **live figure on a client**,
which has to learn its number changed without asking. This is that seam.

⚠ **It is deliberately NOT an `EventApi` broadcast.** The bus is for
genuinely global signals with unknown consumers; this has exactly one
known consumer, so it is a method call. An earlier cut of this build did
mint a bus event per ledger — six classes — and they were not merely
redundant, they were **wired to nothing**: the dependency index cannot
match a durable `templatePath` through a `ChangeSource`. See
[mql-subscription.md](./mql-subscription.md).

⚠ **After the write, never before.** A consumer that recomputes must not
read a ledger missing the row it was just told about.

## ⚠ Seeding the log does not move the figure

`RenownApi.renownOf` reads `RenownStanding.cached()` — an in-memory map
warmed at boot from the **`renown` collection**, which holds the
*materialized* standings. It does **not** read `renown_events`.

So writing rows into `renown_events` (as
`packages/server/scripts/seed-standing.ts` does) changes nothing until a
**recompute** folds the log into the standings collection. A bare
restart re-warms the map from a collection the seeding never wrote, and
is therefore a no-op.

This is the trap that made a correctly-seeded character still read
`dormant` through a whole live drive of the S1 build. The same holds for
`participation` and `producer`. `transcripts` and `disposition_events`
are the opposite — they derive on read straight from the log, so seeded
rows are live as soon as the fold runs.
