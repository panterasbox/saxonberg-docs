# Participation & the consumer influence stock

The **quantity** half of consumer influence — the sibling of
[renown](./renown.md) (the quality half). Renown measures *how valued* you
are; participation measures *how much you show up and take part*. Their
product is the consumer stock's standing: `engagement × renown`, the
cooperative slate's consumer-contribution thesis made into a measured,
banded output.

This build ships the substrate and the projection; it wires **no
consumer** — there is no ballot yet. It exists now because participation is
**time-integrated and cannot be backfilled**: every day not logged is
history lost. So the priority is a *sound raw log* running from day one,
with the scoring as a rebuildable projection that can be re-legislated
without discarding data. See the
polity decision register (D2/D4/D5/D6);
the engagement-meter requirements doc was retired at the pre-merge sweep
and its decisions live in the register.

**Why the product.** Reputation alone is a *popularity contest* (charisma
wins, presence doesn't); engagement alone is an *idle-farm / no-life
grind* (raw hours, bots, AFK); **engagement × reputation** demands both —
substantial participation *and* regard for it (graduated from the
cooperative slate, 2026-09).

## The pipeline (mirrors renown, shares no code)

```
CommandDispatchedEvent ─► participation_events ─► participation ─► InfluenceStanding
   (receive-gated tap)      (append-only log)       (rebuildable      (max(0,renown)
                                                     aggregate)        × participation,
                                                                       banded)
```

- **Capture** — `CommandGiverMixin._emitInputEcho` (the single-fire,
  sensor-gated dispatch tail) fires `CommandDispatchedEvent` once per
  **recognized** command (a verb bound — parse failures carry no `verb`)
  from an **interactive** origin (a real player, never NPC / programmatic /
  cascaded). `ConsumerLogic.installDispatchTap` taps it. Mirrors renown
  tapping `CommReceivedEvent` — the command framework stays ignorant of
  participation.

  **The event's receive side is locked to the `ConsumerLogic` +
  `ProducerLogic` pair** via `EventApi.restrictSubscribe`.
  `command.dispatched` fires on *private* commands too — `query`,
  `settings`, a private `look`, a whisper, a DM, char-gen inputs — none
  in-world-observable. EventApi's subscribe is open by default and `on` is
  author-reachable, so an ungated broadcast would be a per-subject activity
  **snooping** side-channel (any mudlib subscriber watching every player's
  command cadence). Restricting the receive side to the two blessed
  consumers closes that while keeping the bus's producer-ignorant
  decoupling. The labor faucet **shares this same signal** (the event now
  also carries optional `locationTemplatePath`/`actorIdentityPath` for credit
  routing); both taps must assert the *full pair* so an HMR re-assert never
  evicts the other — the clobber trap detailed in [influence.md](./influence.md).

  The mechanism is `EventApi.restrictSubscribe(name, ...consumerClasses)` —
  the first use of the EventRegistry prop-access apparatus's **receive**
  (`Get`) half (emit = `Set` = transmit; subscribe = `Get` = receive). It
  locks an event's subscribe side to an allowlist, tracks ownership by class
  *name* (a hot-reload re-asserts; a different class can't hijack), and now
  also gates renown's `reaction.fired` and `comm.received` taps (which carry
  the same per-actor activity signal).

- **`participation_events`** (`ParticipationEvent`) — the append-only log,
  **one row per active time-bucket**: a member is credited *once per coarse
  bucket* in which they take a recognized action. The append is
  find-or-skip on `{subject, bucket}`. This is **anti-AFK** (idle buckets
  never appear — you must act) and **anti-spam** (a macro spamming a bucket
  earns the one row the bucket already holds), the resolution of the
  slate's "what counts as engagement" question. The deliberate divergence
  from renown: renown appends one row *per signal*; participation dedups
  *per bucket*.

- **`participation` standing** (`ParticipationStanding`) — the rebuildable
  per-subject aggregate: the recency-decayed active-bucket count. A derived
  cache, never authoritative — **drop it, replay the log, get identical
  standings** (the invariant a test asserts). Compact-wide only (no
  scope partition); the `{subject, scope}` shape with `scope = '*'` is kept
  for parity with `RenownStanding`.

- **Projection** — `ConsumerLogic.standingOf` derives, on read,
  `max(0, renownOf(subject)) × participationOf(subject)`, wrapped in a
  `Band`. Both inputs are sync cached reads, so there is no third stored
  source of truth. The `max(0, …)` clamp is **load-bearing** (register D5):
  net-negative regard disenfranchises (zeroes the standing), it never
  drives influence below nothing.

## Two clocks (the divergence from renown)

Renown decays in **game-time** (its signals are in-world social acts).
Participation decays in **real-time** — it measures *a human showing up*,
and "present and contributing *now*" is real now. So:

- the **bucket key** is `floor(realAt / bucketMs)` (wall clock);
- the recompute's recency-**decay** uses `realAt` deltas, real-time
  half-life.

`ParticipationEvent.at` (game-seconds) is recorded for parity/correlation
only. The recompute *cadence* rides real-time `ScheduleApi.recurring`, as
renown's does.

### ⭐ The pass scores who changed; decay is read (db-performance, 2026-11)

Until db-performance W3 the 60 s pass read the whole event log and found
and saved every subject who had ever acted — in a world that had seen 200
players, 3 642 database ops in three idle minutes. Now:

- **a pass re-scores only the subjects whose rows changed** since the last
  one (a module-private dirty set, marked by every append AFTER its write).
  The first pass after a start or a hot reload is full — a fresh module
  holds "everything" — so there is no cursor to persist and no margin to
  get wrong; so is the first pass after the value function changes (a
  fingerprint of the half-life). Nothing changed → **no round trip**. The
  rows are written as ONE `saveMany` keyed on `{subject, scope}` and put
  straight into the read index (no re-warm). A failed pass gives its
  subjects back. `recompute({ full: true })` re-scores everyone.
- **decay is applied when a standing is READ**: the index holds the figure
  and the real-time instant the pass computed it at, and
  `participationOf` carries it forward with one exponential — exactly
  what a pass at the read's instant would store, to rounding, and fresher
  than a figure up to 60 s stale. A subject nobody touches needs no pass
  for its standing to fall. (`ConsumerLogic.passes.test.ts` proves the
  equality and the bands.)

The producer stock (influence.md) runs the same pass, keyed on the author.

## The three-stock contract (symmetry at the output)

The consumer stock is the first of three influence stocks (consumer /
capital / labor). It is built consumer-only, but against a shared
**output contract** so the others slot in without a refactor:

- **`InfluenceStanding`** `{subject, stock, scalar, band}` — the uniform
  output shape every stock emits, carrying the `stock` identity tag (the
  renown-`scope` mirror).
- **`Band`** — a stock-agnostic qualitative tier (register D6: standing is
  shown as a band, never a grindable number; the scalar is reserved for the
  ballot). Cutoffs are an AppSettings dial.
- **`InfluenceApi`** — the thin common dispatcher: `standingOf(subject,
  stock)` delegates `'consumer'` to `ConsumerApi` and `'producer'` to
  `ProducerApi` (both now live — see [influence.md](./influence.md));
  `'capital'` is the one reserved value, returning a defined zero standing,
  never a throw.

The symmetry lives at the standing/band layer; the **asymmetry stays at
the source** — each stock's faucet and formula differ (capital is
`concave($)`, *not* `engagement × quality`, so no shared formula could
fit). Raw logs stay per-faucet.

## Api topology

- **`ConsumerApi`** (`api/consumer.ts`) → **`ConsumerLogic`**
  (`platform/idea/api/ConsumerLogic.ts`, `/platform/idea/api/consumer`) — owns the
  participation faucet (`append` / `eventsFor` / `recompute` /
  `participationOf` / `boot`) and the consumer projection (`standingOf`).
  Reads `RenownApi`; never owns it.
- **`InfluenceApi`** (`api/influence.ts`) — the common cross-stock
  dispatcher; no faucet, no logic singleton.
- **`RenownApi`** — the shared, general social-standing substrate, read for
  the quality half. Untouched by this build (it is dual-projection —
  governance reads the Compact-wide roll-up; NPC/social/disguise read
  the per-circle vector — not a consumer-chamber thing).

Value-objects live in `lib/standing/` (`ParticipationEvent`,
`ParticipationStanding`, `Band`, `InfluenceStanding`) — the consolidated
home shared with renown's value objects and the labor/conviction
substrate (the influence build folded `lib/participation/` + `lib/renown/`
into one `lib/standing/`; renown is *not* subordinate, hence the neutral
name — see [influence.md](./influence.md)). The capture event is in
`lib/events/CommandDispatchedEvent.ts`. Named **Participation** (not
"engagement") to avoid colliding with `lib/activity/`'s in-session
`EngagedMixin` — a different concept (sustained in-session actions).

Standing banks on the giver's durable **identity path**
(`getIdentityPath()` — `/platform/agent/Avatar/<playerId>` for a player,
its own row for an NPC), **not** the ephemeral `stuffId` (re-minted on
re-clone) and **not** its lineage. The `stuffId` stays a live-resolution
handle; only the stored key is the identity.

⚠⚠ **Until db-performance W1 (2026-11) the stored key was the lineage**
(`getTemplatePath()`), and every player body is cloned from one row — so
every player's participation was ONE subject (`/platform/agent/PrimaryAvatar`),
every shade's another, and every reader that asks by identity (`standing`,
the profile, the dashboard) read zero. Those rows were left where they are
(no migration) and decay, read by nobody; each person's participation
starts from the fix. The identity key also makes a shade continue its
person: a living body and its shade are one standing.

**A giver with no identity is skipped** (D3): a login screen's commands
earn nothing. Its raw `stuffId` was a subject no reader could ever ask
for, minted once per connection and re-scored forever.

## Tuning (AppSettings, no code defaults)

- `participation.bucketSeconds` — active-bucket width (real seconds).
- `participation.decayHalfLife` — recency half-life (real seconds).
- `influence.bandThresholds` — JSON `[{name, min}]` band cutoffs
  (stock-agnostic).

The entrenched invariants — the `engagement × renown` form, the
`max(0, renown)` clamp, derive-don't-track / rebuildable cache — are
**code, never keys**.

## The `standing` verb

A zero-arg, self-only, read-only self-view (afforded by `PersonaMixin`,
sibling to `chronicle`). Renders the actor's participation, renown, and
resulting influence **band** — qualitatively, never the raw scalar.

## Deferred (named, not built)

The ballot / chambers / voting (though the **conviction spend substrate** is
now built — `ConvictionApi`, no verb yet; see [influence.md](./influence.md));
the capital faucet (Twitch subs); **second-order engagement** (register D2 —
"the engagement you cause in others", the population-dependent quality
enrichment; this build measures the solo-observable quantity); the
player/human-level rollup for enfranchisement; per-faucet saturation. The
**labor faucet is now built** (the make stock — see
[influence.md](./influence.md)).


## The standing witness

After each append is persisted, the ledger calls
**`SubscriptionApi.poke(SUBSCRIPTION_SOURCES.mql, null, { durableSubject: subject })`** — a direct method
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

So writing rows into `renown_events` by hand changes nothing until a
**recompute** folds the log into the standings collection. A bare
restart re-warms the map from a collection the seeding never wrote, and
is therefore a no-op.

This is the trap that made a correctly-seeded character still read
`dormant` through a whole live drive of the S1 build. The same holds for
`participation` and `producer`. `transcripts` and `disposition_events`
are the opposite — they derive on read straight from the log, so seeded
rows are live as soon as the fold runs.
