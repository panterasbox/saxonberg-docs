# Residency — scheduled object self-maintenance

The **residency** substrate hosts a family of scheduled sweeps that share
one shape: the engine periodically visits objects and lets each decide a
maintenance action on itself — *engine informs, object decides*. Two
sweeps live here:

- **Eviction (shipped)** — a real-time background sweep that lets
  abandoned in-memory `Stuff` **evict itself**, reclaiming the cold tail
  of clones that were made, drifted, and then forgotten (a dropped item
  nobody returns for, a room nobody re-enters, an NPC nobody visits, a
  shadow lifted off its host and orphaned). Before it, the only way a
  live `Stuff` left memory was an explicit `StuffApi.destruct()`.
- **Reset (SHIPPED — same shape, same home)** — a **game-time** repop
  sweep over `ResettableMixin`, restorative-of-self (the eviction
  sibling). `ResidencyApi.boot` installs it alongside eviction
  (`installResetSweep()` on the game-time `WorldClockApi.every` clock, so
  it freezes with a paused world — the eviction divergence);
  `runResetSweep` visits every resettable and lets it restore itself, and
  `residency.reset.*` knobs govern it. See *The reset sweep* below. Only
  the general store's `Stock` is wired as a consumer today.

The rest of this doc describes the eviction sweep, then the reset sweep.

**This is a garbage-culler, not a swapfile.** Memory *scaling* is an
explicit non-goal (the answer to that is a distributive model). The
culler never promises rehydration and never snapshots drift: an evicted
object is simply *gone*, and if it's referenced again the existing
Pattern-A / Pattern-C resolution re-clones it fresh from its template.
That's acceptable by construction — the premise is that nobody cared
about the drift, because nobody touched it. This collapses the hard
"passivate stateful clones" problem entirely: there is no state to
preserve, because we only cull what's already abandoned.

See the seeding residency-slate.md.
(The requirements doc was retired at the pre-merge sweep; this doc is the
record of what shipped.)

## The whole mechanism

Deliberately small: **one method, one signal, one sweep.**

> **The veto roster stays hand-written, deliberately.** The
> reference-lifetime build declares field lifetimes (`weak` /
> `symmetric` / `owned`) and it is tempting to derive the eviction
> vetoes from them — a holder of `owned` targets vetoes, and so on. It
> does not, because the two shipped vetoes that matter both carry
> case-specific carve-outs a mechanical rule would flatten:
> `Container.canEvict` does NOT veto on contents when the host is
> `Persistable` (its contents are captured before the cull), and
> `Exit.canEvict` does not veto for an unbound post-clone exit. Both are
> pinned by regression tests so the build could prove they did not move.

### One method — `Stuff.canEvict`, default cull, override to prevent

The decision is fully inverted: the sweep asks, the object decides.
`Stuff.canEvict(context: EvictionContext): VetoResult` defaults to
`{ ok: true }` — a fresh backing class is reclaimable by default and only
becomes sticky when its author deliberately vetoes (the correct bias for
a leak-plugger). It is a public, ungateable `@hook` (a subclass's
`super.canEvict()` is author code), landing in the **extension** doc
tier. There is **no scope filter and no class-keyed default**; every
`Stuff` is asked the same question. Vetoes layer on the mixin/class that
*owns* the relevant relationship, composed via `super.canEvict(context)`
— base `Stuff` stays permissive and never reaches into
Container/Shadow/Avatar/Exit knowledge.

`EvictionContext` is thin and **extensible** — `{ idleMs, reason }`
today. It carries only runtime facts the object can't self-derive; the
sole reason `canEvict` takes a context rather than being no-arg is so a
future resource-pressure trigger (the property slate's compute-dormancy)
can add fields (`memoryPressure`, …) and a new `reason` without touching
a single existing override.

### One signal — recency (`lastTouched`)

A coarse, transient `lastTouched` timestamp on `Stuff` base (never
persisted, not in `fieldMeta`'s persistent entries; resets to construction time on
every clone/hydrate). `touch()` / `getLastTouched()` are the surface.
The security gate calls `touch()` on the raw target after **every
successful non-getter method dispatch** — being *used* is being touched,
so anything a player or the system exercises stays warm for free, and
only genuinely un-called objects go cold. Denied dispatches don't touch;
getter reads don't touch (a passive read shouldn't keep an object
resident).

**Presence is the strongest form of touch.** A connected player sitting
silently in a room may dispatch nothing, so before each scan a *presence
walk* (`ConnectionApi.getAllInteractives()` → holder → room →
`getDeepContents()`, the `WeatherLogic.runBoundaryFanout` pattern)
refreshes the recency of every occupied room and everything nested in it
(co-occupants, floor items, and — since the player is himself
deep-contents of his room — each player's inventory). This is a touch
*source*, not a pin.

> **The enumeration-is-not-use rule.** `StuffApi.getAllObjects()` checks
> liveness on the **raw** target (`ProxyApi.unwrap(obj).isDestroyed()`),
> not the proxy. Enumerating the registry — which the sweep does every
> tick — must not dispatch-touch every object, or the sweep would refresh
> all recency before reading it and could never cull anything.

### One sweep — a lazy O(n) scan

`ResidencyLogic` (at `/obj/api/residency`, behind the thin gated
`ResidencyApi`) drives a real-time `ScheduleApi.recurring` sweep:

```
presenceWalkImpl()                                  // refresh presence first
for obj in StuffApi.getAllObjects():                // proxies
  raw = ProxyApi.unwrap(obj)
  idleMs = now - raw.getLastTouched()                // RAW: idle check must not touch
  if idleMs < idleThresholdMs: continue              // LRU grace
  if not obj.canEvict({ idleMs, reason: 'idle' }).ok: continue   // PROXY: this-relative vetoes resolve
  observe → log(candidate);  enforce → StuffApi.destruct(obj)
```

The split is deliberate: recency is read on the **raw** target so the
idle check itself is never a touch (that's what keeps idle detection
honest), but `canEvict` is asked on the **proxy** — a veto's
self-knowledge can be `this`-relative through the framework (a shadow's
host lives in a proxy-keyed WeakMap, so `this.host` only resolves when
`this` is the proxy). The touch proxy-dispatch incurs is harmless: only
cold-tail candidates (past the idle threshold) are asked, and only
*after* the idle decision — a culled candidate is moot, a vetoing one is
merely refreshed.

**Not an ordered LRU list.** The textbook LRU structure is for eviction
*on the hot access path* (find-a-victim-now under memory pressure); ours
is a lazy background timer, which flips the tradeoff. An ordered list
would move cost onto the touch path (every dispatch a move-to-front;
presence bursts a whole subtree); a plain scan keeps touch at a single
write and reads `n` timestamps once per sweep — cheaper in aggregate.
Budgeted/incremental slicing, then a true LRU, are drop-in escalations if
`n` ever makes a single scan spike; `canEvict` never changes.

**Eviction is `StuffApi.destruct`, not a new teardown path.** Culling
runs the ordinary choreography (`onDestruct` → mixin `cleanupOnDestruct`
(R2.4) → shadow detach → unregister), so a culled object's live-ref
holders are cleaned/self-healed by the existing R2.x rules — a dangling
Pattern-B ref is a hazard `destruct` already handles, not one eviction
introduces (which is why residency needs **no reverse-ref index**).
`canEvict` and `canDestruct` are independent: an object that permits
eviction may still `canDestruct`-veto, so the enforce path tolerates a
`DestructError` (log + continue).

## Observe-first, then tune

Because the go-live memory profile is unknown, the sweep **ships in
observe mode** — it logs what it *would* cull (cold-tail size + a sample)
and destructs nothing. Watch that against real load, then flip
`residency.eviction.mode` to `enforce`. Mode is re-read each sweep (no
restart). Knobs are per-sweep-namespaced (`residency.eviction.*` here;
the reset sweep will add `residency.reset.*`), seeded in
`config/app-settings.yaml`:

- `residency.eviction.mode` — `observe` | `enforce` (default `observe`)
- `residency.eviction.intervalMs` — sweep cadence (default 60000)
- `residency.eviction.idleThresholdMs` — grace window (default 1800000)

Settings reads are try/catch-guarded and fail **safe** (to `observe`,
never cull) when AppSettings isn't warmed (pre-boot / tests).

## The veto roster

`Stuff.canEvict` base returns `{ ok: true }`. Two kinds of veto layer on:

**(A) Categorical infrastructure.**

- **All `*Logic` singletons**, via a shared **`ApiLogic extends Idea`**
  base whose `canEvict` vetoes unconditionally. Logic singletons are a
  finite, load-bearing set underpinning every system; they'd self-heal if
  culled (`StuffApi.singletonSync` re-warms), but there's no benefit to
  churning them, so the veto is applied once on the base rather than as
  ~65 per-class overrides. `ApiLogic` is the sanctioned home for future
  shared logic-singleton concerns. (`instanceof Idea` still holds.)
- **Stateful registries + `Interactive`** (`ReactionRegistry`,
  `MqlSubscriptionRegistry`, `OfficeRegistry`, the catalogues,
  `Interactive`, …) veto per-instance: they hold irreplaceable in-memory
  state and can't ride `ApiLogic` (not `*Logic`) nor be blanket-protected
  on `PostRegistrationMixin` (also worn by cullable `NPC`).

**(B) Relational / structural — derived from the R2.x ref-cleanup
rules.** The attachment relationships that require coordinated cleanup on
destruct (see [ref-shapes.md](../ref-shapes.md)) are exactly the ones
that require veto-coupling on evict: **an object in an R2.1-owned or
R2.2-symmetric relationship vetoes while its anchor is alive.** This is
self-knowable — the object sees its own side of the edge (its container,
its host, its owner) — so it never reintroduces the rejected reverse-ref
index.

| Host | Vetoes while… | Owns |
|---|---|---|
| `Container` | non-empty **AND not a persistence host** | cold contents cull first; the emptied container culls a later sweep (bottom-up — R2.4 owning-cascade never surprises). A **persistable host** (`PersistableMixin`) falls THROUGH this veto — the sweep `await`s `PersistableApi.capture` before `StuffApi.destruct`, so contents are durably captured then re-materialize on next reference (see [persistence.md § eviction seam](./persistence.md#the-eviction-seam)) |
| `Containable` | inside a `HasInteractive` holder | protects a disconnected-but-in-memory avatar's inventory that presence-touch can't reach |
| `HasInteractive` | always (Avatar / Login) | the session holder itself; lifecycle owned by connection teardown |
| shadow | its host is alive | one-directional (checks `host.isDestroyed()`, never `host.canEvict()` — no recursion); orphaned shadow culls |
| `Exit` / `Adornment` | its source room / wall is alive | R2.1 owned — else an exit is culled out from under a live room |
| `Behaved` | it holds a behavior spec | authored NPC cast; re-clone would erase it |
| `WarrenMember` | it's in a live warren (`getWarren() !== null`) | culling a satellite out from under the elastic graph (host designation, hub exits, migration) is the Warren's call, not residency's |

A plain `Location` does **not** veto. Rooms are path-addressable, so an
empty, unreferenced, idle room culls and simply re-clones from its
template on next entry (exits carry `destinationPath` to re-resolve). A
room with contents is held by the `Container` veto; a room in the Warren
graph by the `WarrenMember` veto — nothing else about a room is worth
keeping resident.

Occupied furniture and engaged actors get **no explicit veto** — the
occupant is present, so presence + dispatch touch already keep them warm.

## Module homes

- `api/residency.ts` — `ResidencyApi` gated forwarding shell (`boot`
  installs the residency sweeps; `evictNow` for test/manual).
- `obj/api/ResidencyLogic.ts` — the `@internal` logic singleton, home to
  the scheduled sweeps: `installEvictionSweep()` + `runEvictionSweep` +
  the presence walk today, one retained handle per sweep (the reset
  sweep's `installResetSweep()` + `runResetSweep` land alongside).
  `extends ApiLogic`, so it self-exempts.
- `lib/stuff/Stuff.ts` — `canEvict`, `EvictionContext`, `lastTouched`,
  `touch()` / `getLastTouched()`.
- `lib/stuff/ApiLogic.ts` — the logic-singleton base carrying the
  categorical veto.

## The reset sweep

The second sweep this substrate hosts (SHIPPED) — a **game-time repop**
scan, restorative-of-self, never destructive-of-others. It shares the
eviction sweep's home (`ResidencyLogic`), the presence walk, and the
observe/enforce discipline, but diverges on three axes:

- **Clock.** `installResetSweep()` rides `WorldClockApi.every` (game
  time), not `ScheduleApi.recurring` (real time) — so restock freezes
  with a paused world (the employment-tick / crafting precedent). One
  retained `ClockHandle`; re-install is a no-op.
- **Predicate.** Not eviction's idle + `canEvict`; instead
  **presence-SKIP + object-override**. `runResetSweep` visits every
  `MixinApi.isResettable(raw)` object and, if it sits (at any depth) in a
  room a player occupies, **skips** it — *unless* the object opts in via
  `resetsWhilePresent()` (the shop restocks while browsed). To share the
  present-room set, `presenceWalkImpl` now **returns** the visited-room
  `stuffId` set (the eviction sweep ignores the return; reset reads it).
- **Default mode.** `residency.reset.mode` ships **`enforce`** (repop is
  additive and safe), where `residency.eviction.mode` ships `observe`
  (culling is destructive). `residency.reset.intervalS` is the game-time
  cadence.

`ResettableMixin` (`lib/residency/Resettable.ts`) is the marker + the
two-decision contract (engine informs, object decides): `reset()`
(restorative-of-self, **never a money leg** — Law 2) and
`resetsWhilePresent()` (default `false`). The **first and only wired
consumer** is the general store's `Stock` (see [retail.md](./retail.md)):
its `reset()` tops each stock line back to authored par by cloning fresh
goods (items, never coin), and it overrides `resetsWhilePresent() → true`.
The `Stock` also self-stocks on standup by calling `reset()` from
`postRegister`, so the same one mechanism handles boot-stock and ongoing
topup. (Note the subsumption: a *cold* resettable object is simply culled
and re-cloned fresh by eviction — its template state *is* its reset state
— so explicit reset only earns its keep for **warm/resident** objects
like a browsed shop.)

Other reset consumers (respawning NPCs, resource nodes, container/door
repop) are deferred — the sweep ships complete-at-tier, only the shop is
wired.

## Deferred

- **Note — first elastic-content consumer:** the leased dorm room
  dorms-when-empty over exactly this eviction seam (its `DormWarren` reaps an
  empty room capture-then-cull, and runtime floor corridors reap top-down);
  see [residence.md](./residence.md).

- **Memory-pressure-driven aggressiveness** — a sweep-side threshold
  modulation (shrink the grace window under heap pressure), *not* an
  input to `canEvict`. Documented seam; not built.
- **Per-object retained footprint** — a future `EvictionContext` field
  for the property slate's compute-dormancy trigger (a resource-pressure
  eviction that rides this same `canEvict` + `destruct` seam). True
  retained size needs a heap walk we can't afford per-sweep.
- **Ordered LRU / incremental sweeping** — escalations if observe-mode
  data ever shows the O(n) scan spiking.

## The presence walk and the sandbox boundary

`presenceWalkImpl` runs from a field root and reads each connected
holder's room — but a holder may be a wire body standing in a circle,
which is a cross-boundary dispatch. Residency spans the boundary **by
definition**: it keeps alive whatever is in use, and a room someone is
standing in must not be culled out from under them, circle or field.

The per-holder body therefore goes through
`SecurityApi.projectAcross`. Skipping circle holders instead would not
do: an uncaught deny aborts the whole walk at the first circle
occupant, so one player stepping into their own circle silently turned
off residency keep-alive for the entire world. See
[sandbox.md](./sandbox.md).

## The corpse joins the veto roster (2026-07-31)

`PostmortemMixin` vetoes `canEvict` while a dead body has not finished
decaying — "a body still lies here". At the terminal `spent` stage it stops
objecting rather than destructing itself: **withdrawing an objection**, so
the ordinary sweep decides, and goods on the corpse evacuate through the
shipped container behaviour instead of dying with it. See
[mortality.md](./mortality.md).

Note the eviction sweep ships in **observe** mode, so a spent corpse
permits collection but is not actually culled in production; the contract
is tested directly.

## Zone fields the spawn sweep reads

All three come off the ordinary `zone.lookupField` walk, so a parent
zone's declaration covers its descendants and a child can narrow it.

| Field | Effect |
|---|---|
| `stocks` — `{censusKey: count}` | overrides an item's own `regionTarget` |
| `favours` — material tags | multiplies draw weight for matching items |
| `blessingOdds` — BUC weights | **overrides an item's own generation odds, zone-wide** (see [magic-items.md](./magic-items.md) § Generation odds) |

⚠ **None of the three is declared by any shipped zone yet.** A region
that declares nothing leaves every item on its own baseline and a
neutral affinity, so distribution works in un-authored regions rather
than silently placing nothing.

The BUC roll fires on the freshly cloned object at the mint site inside
the sweep — deliberately **not** in `StuffApi.clone`, so an author's
clone, a crafted output and a restocked consignment never roll.

## ⚠ A second installed sweep

`CardApi.boot` installs one recurring `ScheduleApi` callback that evicts
the cold tail of the **card set** — the same shape as the eviction sweep
here, for a different population. See
[card-surface.md § One sweep, and exactly one clock](./card-surface.md).

Worth cross-referencing for two reasons:

- ⚠ It is a **second** installed sweep, so the pattern is now a pattern
  rather than a one-off. A third wants the same three properties: one
  handle for the whole population, a coarse cadence with a fine window,
  and an explicit principal.
- ⚠ **The principal is re-planted.** A scheduled callback fires long
  after the frame that installed it, so the execution context has no
  target and a gated registry call would be denied on every tick —
  silently, because a scheduled callback has nobody to report to. The
  card sweep wraps itself in `ExecutionContextApi.runRoot` with its
  logic singleton as principal; the eviction sweep reaches only
  ungated surfaces and does not need to.
