# Attendant — the storefront-attention substrate

The universal "walk into a storefront, wait, get attended one at a time"
subsystem. Every service venue (the bar, the bank, the TPA ticket office,
future shops) runs it, configured differently — it is **not** a bank feature;
it is foundational and sits underneath everything. Lives in `lib/attendant/`.
Graduated from ../staging/attendant-subsystem.md.

## The model

An **attention relationship**: a *server* attends *customers* one at a time; the
rest wait in an *order* and are **poked when it's their turn** — never frozen at
a modal "please wait" (the keystone that keeps a universal queue from being
anti-fun; you mill about, you don't stare at a spinner). Being-attended unlocks
a venue-specific **payload verb** (`order` / `deposit` / `buy`). **The subsystem
is the wrapping; the verb is the payload** — bar, bank, ticket office are the
same wrapping around different payloads.

- **`AttendantMixin`** (`lib/attendant/Attendant.ts`) — the capability, hosted
  on a service-point `Thing` fixture (the `BankMixin`/`Menu` precedent: a
  Location's own `commandContributions` don't reach its occupants, so the
  counter is a fixture). Owns the **queue** + the active **leases** + the
  per-venue config. `AttendancePoint` is the bare concrete class; a real venue
  composes the mixin onto its own fixture.
- **`AttendanceEngagement`** (`lib/attendant/AttendanceEngagement.ts`) —
  being-attended as a `SustainedEngagement` occupying the **server's**
  `attention` slot. "One at a time" *falls out of the slot*: a server has one
  attention slot, so it holds exactly one attendance; a second customer waits.
  The customer side is deliberately NOT slot-locked (you're free while you wait
  and while you're briefly served — a MUD, service is async); the exclusive
  resource is the server's attention, and this engagement IS the lease on it.
  Instant venues (attendDurationMs 0 — the bar) never create one: the payload
  runs synchronously, holding no slot.
- **`Ticket`** — the take-a-number claim (a carried `Thing` stamped with its
  point + number). The functional line-vs-ticket difference is one thing: must
  you stay present to hold your place? A line holds it by presence; a ticket
  holds it as a carried object.

## Config axes (how venues differ on the one substrate)

Authored on the seed, read by the substrate:

- **discipline** — `reception` (recognized/newcomers received, priority skip;
  the bank) / `line` (FIFO, presence holds your place) / `take-a-number` (a
  Ticket holds it) / `scrum` (no order, the server gets to you; the bar) /
  `appointment`.
- **serverPositionKeys** — which employment `Position`s attend (servers =
  on-shift employees, resolved via the employment roster — reuse).
- **staffingPolicy** — `close` (shuttered when unstaffed; the card still works
  elsewhere, never a lockout) or `self-service` (a bounded machine, no human).
- **attendDurationMs** — 0 = instant (a soda) / a real dwell (a loan talk).
- **skin** — the diegetic strings (the poke, the eviction nudges): the anti-grief
  *is* the characterization.

## The two anti-grief guards (completeness requirements, not options)

Handing players an exclusive or shared resource in a multiplayer world is
inherently griefable; each pattern has its standard guard (the
`anti-grief-resource-guards` memory). The substrate ships **with** them.

- **Exclusive resource → a LEASE** (`AttendantLogic`, `api/attendant.ts` +
  `platform/idea/api/AttendantLogic.ts`). You hold the server's attention only while
  *actively using it*. **Recency** is a real-time stamp on the lease, bumped by
  every service act (`bumpLease`); a **lazy real-time sweep** (`ScheduleApi.recurring`,
  mirroring `ResidencyLogic.installEvictionSweep` — griefing is a real-time act,
  so the watchdog is real-time, not game-time) finds stale leases and **aborts**
  them (`service-idle` `AbortReason`) → the holder is bumped, the next customer
  pulled up. **Default-evict** — the griefer gets no veto; the venue configures
  the timeout, never *whether*. A service act resets the counter, so a
  *legit-slow* customer (thinking, typing) is distinguished from an *idle
  griefer*. **Linkdead = instant release** (subscribe `PlayerDisconnected`).
  `attendant.lease.*` / `attendant.queue.*` AppSettings.
- **Common-pool resource → a QUOTA** — the sibling guard, first worked at the
  bank till (a per-account cash-withdrawal cap; see [banking.md](./banking.md)).
  Per-actor only, never collective (a bank run is a feature).

Not lifted into a shared primitive yet — Attendant owns its lease, built clean
to lift later.

## The pieces, in Stuff terms

**The venue's behavior lives on the mixin, not in an Api.** The point owns its
queue and leases, so it owns the verbs over them: `AttendantMixin.requestAttention`
(grant a lease on a free server / queue / instant / closed / self-service),
`release`, `bumpLease`, `serveNext`, `leaveQueue`, `evictIdleLeases`,
`dropCustomer`, plus its own `resolveServer` (roster → present, on-shift,
attention-free). A controller calls `point.requestAttention(key)` directly —
it's the venue's behavior, not an Api's, and there's no privilege gate to route
through (an early design assumed one, but the queue/lease state is transient
in-memory order, not persisted or economic — the real guards are the sweep and
the till/quota). `Mixins.Attendant` / `MixinApi.isAttendant`.

The **one genuinely Api-shaped piece** is `AttendantLogic` (`/platform/idea/api/attendant`,
behind the thin `AttendantApi`): a process-level **background service** — the
real-time idle-eviction sweep (a retained `ScheduleApi.recurring` handle) + the
`PlayerDisconnected` subscription + the `boot` seam. It's the ResidencyLogic
shape: the engine drives cadence and **informs** each point (`point.evictIdleLeases(now, threshold)`
/ `point.dropCustomer(key)`); the **point decides**. Booted after employment
(server resolution reads on-shift state).

## Instances

- **The bar** (`/world/lounge/thing/bar-counter`) — scrum / zero-wait: you don't queue
  at a bar, the bartender gets to you. `OrderController` consults the point
  before serving; instant, so behaviorally unchanged — it runs the substrate and
  earns the lease.
- **The Goodkin bank** — reception; its being-attended for enrollment reuses the
  `DialogueConversation` engagement (the officer *receives* you), while quick
  service (a plain deposit) is instant. See [banking.md](./banking.md).
- **The TPA ticket office** — a formal counter (retrofit deferred; identical in
  shape to the bar).

## Deferred seams (named)

The fairness-cap-under-contention (active-hogging beyond idle + re-queue-to-back);
appointment/scheduling beyond FIFO + ticket; the full take-a-number "now serving"
dynamic-Detail board + ticket idle-expiry wiring; the lifted shared
exclusive-lease primitive; player-run/player-staffed points; a client
queue-position card; the self-service machine's cash ops.
