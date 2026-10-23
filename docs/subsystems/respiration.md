# Respiration

Respiration is the body's **air-exchange substrate** and the acute crisis
when that exchange stops (**asphyxiation**: drowning, vacuum). It is the
vitals *driver* that **drives [`spo2`](./vitals.md)** — a live, banded
vital sign (`Quantity<'%'>`, baseline 98, survivableMin 70) that gates
consciousness. Since the underwater build it is also the body **under
the water**: the breath as a store, the urge to breathe as a second clock
that can lie, cold shock, the ears, the bends, the tread that keeps a
swimmer up, and the drowning response. Where the water itself lives —
the column, the crossings, the medium, the diving trade — is
[underwater.md](./underwater.md).

Unlike the lazy reconcile-on-read drivers ([metabolism](./metabolism.md),
thermal), respiration is an **event-triggered scheduled-engagement crisis
driver**: benign ambient runs *nothing* (no scheduler, no `spo2`
movement, zero reconcile — the idle case is provably zero-cost); a
hostile-medium event engages **one** bounded, cancellable drain that
self-terminates the instant air exchange resumes. Respiration was the
first concrete engagement producer in the codebase — the
[activity](./activity.md) framework's first real `SustainedEngagement` +
`ScheduledEmission`.

Everything lives on one mixin — **`RespirationMixin`**
(`lib/respiration/Respiration.ts`) — plus its colocated engagement
classes (`lib/respiration/RespirationDrain.ts`: `RespirationDrain`,
`RespirationRecovery`, `RespirationTread`). There is **no
`RespirationApi`**: every caller is the host body itself. The dials are a
module const-object (`RESPIRATION_DEFAULTS`); **rates are playtest-tuned,
not engine invariants**.

## Compose, don't absorb

Respiration owns **no-oxygen → `spo2`**, and the body's side of being in
water (breath, ears, gas, tread). Cold → thermal; toxin → metabolism's
toxin-burden; opacity → perception; the medium's physics →
`lib/biome/Medium.ts`. It does **not** read `MetabolicMixin.spo2Throttle()`.
**Plumbing, not anatomy**: it reads channel-state (exchanging or not),
never orifices. The `respires` opt-out on the body plan opts a body out
entirely.

## Composition

`RespirationMixin` composes on a `Creature`-shaped host immediately outer
of `MetabolicMixin`. The host cast is `Stuff & Vitals & Organism`. The
crisis engagements need `EngagedMixin` and the move-reassess needs
`MobileMixin` — Character-tier — so those surfaces are **runtime-narrowed**
at their use sites. A body without an `Engaged` slot resolves the medium
but holds no scheduled drain (a documented degenerate; give it
`EngagedMixin` to make it drownable).

## ⭐⭐ The respiration clock is REAL seconds

`respirationNowSeconds()` is the world clock's game-seconds **divided by
the world scale** — real seconds. A held breath is something the player
does at the keyboard: on the game clock a full breath at 12× was six real
seconds, not time to type one command, and two divers blacked out before
they could speak (the underwater drive). Every respiration interval reads
this clock, so dividing it once makes all of them human-time: the store,
the urge, the hyperventilation window, cold shock, the deep-breath hold,
the drowning response, the ears, the gas compartments, the tread, the
tank tap and the `spo2` drain and recovery rates. The engagements' beat
is real-time too (`EMISSION_INTERVAL_MS` × the scale — a beat every two
real seconds). The charged shell's standby draw is the precedent.

Two hand-offs cross back to game time, and each is scaled up by the world
rate on the way out:

- ⭐ **The dying window.** `DYING_WINDOW_SEC` (90, × the cold factor) is a
  real-time window, but the [mortality](./mortality.md) dying clock counts
  game-seconds — handed over raw it was 7½ real seconds at 12×, too short
  for any rescue. `reconcileAnoxiaCascade` passes
  `DYING_WINDOW_SEC × coldWindowFactor × scale` to `beginDying`.
- The **timed crossing** a swim down the water takes (underwater.md):
  real seconds, run as a game-time step scaled by the world rate.

⚠ **The effort finding (recorded, not changed).** The burn reads
`Exerting.currentPowerW()`, which fades over `CURRENT_POWER_FADE_S` (30)
**game** seconds — 2.5 real seconds at 12× — while the breath reads real
seconds. So effort barely moves the burn: a kicking diver burns little
more than a still one, and the timed swim's effort does not raise it at
all (the step carries no `effortW`). The fix is a real-time fade or an
effort term on the crossing; it is a known gap.

## Determining the medium — the breathable read

Whether a body exchanges gas turns on what it is breathing *from* and how
much of its need that medium meets.

⭐ **Breathing is a rate** (underwater D21). `BodyPlan.breathing:
BreathingSpec[]` — `{ medium, apparatus, intake }`, intake 0..1 of basal
need — with per-stage overrides on the species
(`Species.breathingByStage`: the bullfrog on gills as a newborn, lungs and
skin as an adult). `getBreathableMedia()` is derived: the media with
intake ≥ 1. A partial intake (a frog's skin under water) slows the drain
by what it takes in; it does not stop it. Gills in water take the
water's dissolved oxygen against the stage's habitat tolerance
(`habitatByStage`), so a trout in foul, oxygen-poor water breathes, and
not enough. A held working (`breathe-water`) confers a medium at intake 1
through `setConferredBreathing`.

**`resolveCurrentMedium()`** reads the one medium read,
`BiomeApi.mediumFor(self)`:

- an engaged locomotion mode whose medium the body breathes wins;
- ⭐ **Swimming is at the SURFACE** (maritime): in a water place's surface
  room your head is out — you breathe the atmosphere over it;
- **under** the surface (a band of the column, a vessel's liquid) the body
  is **immersed** in the substance and breathes it or nothing;
- otherwise the atmosphere resolved outward from the body.

The **known-medium gate**: a medium not in the body's set is hostile only
if it is a known atmosphere (`BiomeApi.breathableOf` throws on unknown);
an immersion is exempt from the exemption — a man in a vat of ale
breathes nothing whether or not the table names ale.

⭐⭐ **A breathable TAG is not breathable AIR (the fire build).** The tag
says *what medium this is*; `BiomeApi.hasBreathableShare(contents)` says
whether enough is LEFT — a **displacement** figure (`1 − contentsSum`
against `fire.air.breathableAirShare`, 0.76), never a toxicity one. ⛔ The
first answer reasoned it from CO₂ toxicity (0.95), which made the canary
react to firedamp — the one design whose point is that the bird answers
a *different* question. `applyMediumContents()` folds a gas row's own
`toxicity` × fraction into the toxin burden, so the mechanism is there
the moment a CO₂ row authors a dose. The diving bell's air pocket fouls
through exactly this floor.

### Immersion — the liquid you are inside (fishing A3)

The nearest containing vessel with a non-empty bulk `interior` answers
with its material: a carp in a bowl breathes `water`; a man in a vat of
ale breathes nothing. `onMoved` re-checks the medium on every containment
move, so a landed fish starts drowning in the hand and stops in the bowl.
`reassess` guards a proxy destructed mid-drain (a released fish). Detail:
[fishing.md](./fishing.md).

## ⭐⭐ The breath is a STORE, burned

The breath is liters of usable oxygen, not a grace timer (underwater
D10). `_heldBreathUntil`, `INHALE_HOLD_SEC` and `UNPREPARED_GRACE_SEC` are
gone; the reflex hold is the store itself.

- **Full store** = lung volume × `USABLE_O2_PER_LUNG_L` (0.075). Lung
  volume is the plan's `lungVolumeL` (default `LUNG_VOLUME_L`, 6) × the
  body's **size factor** (mass over the plan's base mass, clamped
  0.05–4). Gills hold no breath: a body that does not breathe air has a
  zero store.
- **Burn** (L/s) = `BASAL_O2_L_PER_S` (0.3 L/min) × **effort**
  (`1 + currentPowerW / EFFORT_W_PER_DOUBLING`, 100 W) × **conditioning**
  (the wind band: untrained 1.2 … expert 0.8) × **cold** (0.8 in water
  below `COLD_SHOCK_K` — the dive reflex) × the **size factor** (a small
  body burns as little as its lungs hold, so a frog's breath lasts as
  long as its size allows). The drain spends `burn × (1 − intake)`.
- **What you go under with.** Ordinary breathing keeps a **tidal** breath
  (`TIDAL_FRACTION`, half); an `inhale` in the last `DEEP_BREATH_SEC` (30)
  keeps a **full** one. That keeps *prepared outlasts unprepared* true
  from physics (tidal ~37 s, deep ~75 s for an untrained 70 kg adult at
  rest). Breathing refills an emptied store over `REFILL_SEC` (10).
  `exhale` leaves the residual (`RESIDUAL_FRACTION`, 0.2) — and empties
  the lungs, so you sink.
- **`_lungFill`** — the gas in the lungs, tracked apart from the oxygen in
  it (a held breath does not shrink as its oxygen is spent; the CO₂ takes
  its place). `getLungAirL()` is what the water reads for buoyancy.
- **Store at 0** → `spo2` drains at the cause's rate → the blackout,
  **with no cue before it**. Under water the hold is silent: the drain's
  *"You can't breathe!"* is skipped, and only the urge speaks.
- `inhale` where there is nothing to breathe refuses: *"There is nothing
  to breathe here."*

### ⭐⭐ The urge — a second clock that can lie

`_co2Urge` (0..1) rises against **this hold's** store (`_holdStoreL`, set
when the drain arms), reaching 1 as the store you went under with runs
out — not against a full breath's, or a cold-shocked half-breath never
reached its first word. It is spoken in the body's own voice
(`self.body`) at three crossings, one line each: 0.5 *"You want to
breathe."* · 0.75 *"Your chest is burning."* · 0.9 *"You can't hold it
much longer."* You feel the urge; you never feel the oxygen running out.

**Hyperventilation.** An `inhale` within `HYPERVENTILATE_WINDOW_SEC` (10)
of the last is quick: no more oxygen than one breath holds, and the next
hold's urge **starts lower** — by `HYPERVENTILATE_URGE_STEP` (0.2) per
quick breath, down to `HYPERVENTILATE_URGE_FLOOR` (−0.6). The warning
comes later than the oxygen runs out: the shallow-water blackout. ⚠ The
first `inhale` is not quick, so three in a row start the urge at −0.4
(one warning still comes, at ~90 % of the hold) and four at −0.6 (past
the blackout: silent). Recorded as a known gap in underwater.md.

### Cold shock

On the transition into water colder than `COLD_SHOCK_K` (288 K), at the
surface or under it, the gasp: the store is cut to **half a full breath**,
the urge set to at least 0.6, and *"The cold takes your breath."* For
`COLD_SHOCK_SEC` (60) after, every refill and every `inhale` is capped at
half — and ⭐ `inhale` says so: `InhaleResult.shallow` reports a shallow
breath instead of a deep one (a diver told "a deep breath" went down on
half of one and blacked out on the bed). Once a minute at most.

## ⭐ The ears

The ears clear **on their own**, like breathing — there is no
`equalize` verb (a verb is for a choice, never for upkeep). Each reassess
compares the body's depth (the band's, under the surface) with the last:

- **going down** adds the step to `_pressureDebtM`, less the share the
  ears clear on the way by diving band (untrained ¼, novice ½, competent
  ¾, proficient 0.9, expert all);
- **not going down**, the debt clears at `EARS_CLEAR_MPS` (0.5 m per real
  second) — including the seconds a timed crossing took;
- **going up** eases it by the meters climbed;
- a body out cold or in the drowning response clears nothing.

At `SQUEEZE_AT_M` (3 m) of uncleared pressure: the `squeeze` condition
and *"Your ears ache with the weight of the water."*; at twice it,
`burst-eardrum` and *"Something gives in your ear… and the water
spins."* The choice left is how fast you go down. (In the shipped world
the waters are too shallow for a burst: underwater.md, Known gaps.)

## ⭐⭐ The bends

Three tissue compartments with Haldane half-times `GAS_HALF_TIMES_S`
300 / 1200 / 3600 — seconds **on the respiration clock** (real) — each
relaxing toward the pressure the body breathes at: `1 + depth/10` atm
while a **worn supply** feeds it at depth, the surface's 1 otherwise. A
free diver's lungful never loads; neither does breathe-water, nor a bell.
`getGasLoad()` reads the compartments.

The bends are a **surface event**: arriving where the body breathes at
1 atm with any compartment over its M-value (`GAS_M_VALUES` 1.4 / 1.25 /
1.12) afflicts `decompression-sickness` (law `stage`, 20 game-minute
stages, resolved by medicine). **Recompression** — going back down under
enough pressure that no compartment is over — relieves it. The dive
tables print the stops; the stop is dwelling in a band. On the 6 m bed
the fast compartment crosses its limit after roughly eight real minutes.

## Treading, and the drowning response

At the surface of deep water, exchanging, every beat is a
**`RespirationTread`** (the third body-slot engagement): `exert` at
`TREAD_W` (150) × technique by swimming band (untrained 3.0, novice 2.0,
competent 1.3, proficient 1.0, expert 0.8) × buoyancy (0.5 when you
float, up to 4 when heavy) × (1 + the drowning bodies clinging to you),
crediting the swimming Discipline. The recovery tick treads too.

- **No cost** while floating (`float()`: at the surface, at least novice,
  not more than `FLOAT_SUPPORT_N` heavy — reached by `lie` on the water),
  while held up (`holdUp`), or while something floats you
  (net lift ≥ `FLOAT_SUPPORT_N`, 30 N, × the size factor).
- **Spent** → the **drowning response** (`isInDrowningResponse()`): the
  swimmer's own line and the bystanders' scene **once per response**
  (debounced over `DROWNING_RESPONSE_SEC`, not on every beat a tired
  swimmer dips back into it); the voice refuses; the body grabs the
  nearest swimmer at the surface (`beClungTo`).
- **`DROWNING_RESPONSE_SEC` (60) spent** → the water place's
  `takeUnder`, and *"The water closes over your head."*

## The disconnect hold

`holdUp()` / `releaseHold()` / `isHeldUp()` — a **breathing state**, set
by `WaterPlace.holdAtSurface` when a player drops connection in the water
or the flood reaches a linkdead body, cleared by `Avatar.enter()`. Held,
the body is face-up and exchanging at no cost. It is respiration's
because respiration is what reads it (an Avatar field would be a
shape-read across mixins). Leaving the surface clears it, along with
floating and the drowning response.

## The crisis engine — one engagement, lifecycle-parameterized

The central entry is the async **`reassess()`**: resolve the exchange
state, then engage or cancel. It is idempotent, guards the dead and the
non-respiring, and cancels the opposite engagement before starting one
(drain, recovery and tread share the `'body'` slot). Along the way it
runs cold shock, the store refill, the contaminant fold, the ears and
the gas, and warms the swimming and diving bands the hot ticks read.

- **engage** — "not exchanging" starts a **`RespirationDrain`** with a
  cause: `'medium'`, `'supply'` (a worn supply present, with or without
  air) or `'channel'` (accepted, never fired — no struggle substrate).
- **tick-drain** — re-stamp the clock (the linkdead / far-past freeze,
  metabolism's guard), skip silently when the body is actually at the
  surface (a tick between arrival and reassess is not a crisis), **burn
  the store first** (and raise the urge), then tap a worn supply, then
  `spo2 −= DRAIN_PER_SEC_BY_CAUSE[cause] × (1 − intake) × Δt`.
- **cross `survivableMin`** — consciousness flips to unconscious on the
  vital; the blackout cue fires once; a blackout under water is marked
  for the aspiration that follows a rescue.
- **fatal floor** — `spo2` dwelling at/below `ANOXIA_FLOOR_PCT` for
  `ANOXIA_LETHAL_SEC` → `beginDying(cause, window)` (the scaled real-time
  window above, **× `coldWindowFactor`**: 1 at 288 K rising to 4 near
  freezing — about six minutes in ice water). The drain cancels.
- **cancel-on-resume** — "exchanging" cancels the drain and starts a
  **`RespirationRecovery`** that raises `spo2` at `RECOVER_PER_SEC`, refills
  the store, zeroes the urge and self-cancels at baseline; at the surface
  of deep water the tread then takes over.

### ⭐⭐ Air rescues a breath death

In recovery, once `spo2` is back above `survivableMin`, a body whose
dying record is the breath's own — `Vitals.getDyingCause()` is
`asphyxiation` or `strangulation` — is **stabilized**. Without it a diver
pulled out on the hose inside the window stayed unconscious on the quay
(the dying record outranks the blood's oxygen) and would have died there.
Rescued, not healed: a body that blacked out under water and comes round
is afflicted `aspiration` (*"water in the lungs"*: cough → fever →
drowning in bed, by game-hour stages, cured by medicine).

The `asphyxiation` affliction is found-or-afflicted while `spo2` dwells at
the floor and relieved on recovery above it.

## The reassess triggers

1. **Traverse** — `onTraversed(exit)`.
2. **Containment move** — `onMoved(from, to)` (fishing D8).
3. **Periodic re-resolve** — each drain/recovery/tread beat ends with a
   fire-and-forget `reassess()`, so a resurface or a tank refill ends the
   crisis without a move.
4. **A conferred medium changes** — `setConferredBreathing`.

⚠ An in-place atmosphere flip (a room becoming water under a standing
body) is still not a trigger; the water's flood answers its own case by
reconciling its occupants (underwater.md). See
`docs/slates/tails/respiration-slate.md`.

## Voluntary breath control — `inhale` / `exhale`

Object-free posture-category verbs (not `hold breath` — `hold` is a
contended item verb), afforded on the self surface through
`RespirationMixin.commandContributions`, gated `requiresAnimate` +
`requiresConscious`. `inhale` fills the store to a full breath where
there is air (hyperventilating when quick; shallow when cold-shocked);
`exhale` empties the lungs to the residual — you sink, at the cost of the
air you held.

## The carried supply — the tank, the helmet

A worn **`AirTank`** (`platform/thing/AirTank.ts`) carries an `interior`
[bulk](./bulk.md) of `air`; the hard-hat **helmet** is an `AirTank` row,
so the suit and the scuba tank share one supply path. When the medium is
unbreathable a worn supply makes the cause `'supply'`, and the drain tick
taps it — **`TANK_TAP_PER_SEC` = 1/30 unit per real second × (1 +
depth/10)** (Boyle: each breath at depth is denser air) — holding `spo2`
while it has air and falling through to the drop when it runs dry. At
1/30 the generic six-unit tank lasts about three minutes at the surface
and a little over two on the cistern's bed (it was 0.5, tuned for the 12×
game clock — twelve real seconds once the clock went real). A breathable
ambient is checked first, so a tank is never burned in clean air. Refill
is `BulkableApi.transfer` (the dive-shop compressor) or, for the helmet,
the crewed air pump (underwater.md) — the helmet holds two units, about a
minute unpumped, so a hard-hat diver lives on the pump. `AirTank.getAirGauge()` is the fill
fraction (an MQL-projected field; live push on debit is deferred).

## Presence freeze + persistence

Every beat inherits metabolism's **linkdead / far-past freeze** (re-stamp,
integrate nothing). The only persistent field is `respirationClockStamp`.
Everything the underwater build added — the store, the urge, cold shock,
the ears, the gas load, the tread state, the hold — is **runtime**: a
reloaded body has been breathing. `spo2` rides Avatar save/restore.

## Dials — `RESPIRATION_DEFAULTS`

`EMISSION_INTERVAL_MS`, `DRAIN_PER_SEC_BY_CAUSE`, `USABLE_O2_PER_LUNG_L`,
`BASAL_O2_L_PER_S`, `LUNG_VOLUME_L`, `RESIDUAL_FRACTION`, `TIDAL_FRACTION`,
`DEEP_BREATH_SEC`, `REFILL_SEC`, `HYPERVENTILATE_WINDOW_SEC`,
`HYPERVENTILATE_URGE_STEP`, `HYPERVENTILATE_URGE_FLOOR`, `COLD_SHOCK_K`,
`COLD_SHOCK_SEC`, `EFFORT_W_PER_DOUBLING`, `TREAD_W`,
`DROWNING_RESPONSE_SEC`, `FLOAT_SUPPORT_N`, `SQUEEZE_AT_M`,
`EARS_CLEAR_MPS`, `GAS_HALF_TIMES_S`, `GAS_M_VALUES`, `RECOVER_PER_SEC`,
`ANOXIA_FLOOR_PCT`, `ANOXIA_LETHAL_SEC`, `MAX_REASONABLE_GAP_SEC`,
`TANK_TAP_PER_SEC`, `DEATH_CAUSE_BY_CAUSE`, `DYING_WINDOW_SEC`; plus the
module tables `URGE_CUES`, `WIND_BURN`, `VERTICAL_MPS`, `SWIM_TECHNIQUE`,
`EARS_CLEARED_ON_THE_WAY`. All playtest-tuned. The const-object's header
says it: every time is read on the real-second clock, and the one
game-time dial is the tick cadence (`EMISSION_INTERVAL_MS`).

## Deferred

- **Strangulation / the channel trigger** — accepted, never fired.
- **Narcosis** — not built (nothing that ships is deep enough).
- **The effort term** — see the effort finding above.
- **The airlock / in-place atmosphere flip.**
- **Altitude, CO₂ toxicity, radiation** — later fidelity layers.
- **The aerobic coupling** — metabolism's read of `spo2`.
- **The general disconnection rule** — physiology-slate; the water's hold
  is the water-only answer.

## Files

- `lib/respiration/Respiration.ts` — `RespirationMixin`,
  `RESPIRATION_DEFAULTS`, the `Respiration` interface, `InhaleResult`.
- `lib/respiration/RespirationDrain.ts` — the three engagements, the
  `RespirationCause` type, the real-time beat.
- `platform/thing/AirTank.ts` — the worn tank + the gauge.
- `platform/idea/species/BodyPlan.ts` (`breathing`, `lungVolumeL`,
  `voice`, port media), `platform/idea/species/Species.ts`
  (`breathingByStage`, `habitatByStage`).
- Condition rows `platform/idea/Condition/respiration/{asphyxiation,
  aspiration,squeeze,burst-eardrum,decompression-sickness}.yaml`;
  Discipline rows `swimming`, `diving`.
- `cmd/posture/{inhale,exhale}.yaml` +
  `platform/idea/cmd/posture/{Inhale,Exhale}Controller.ts`.

## History

The first build (`feature/respiration`) shipped the crisis core and the
carried supply. The underwater build (MR !408) replaced the grace timers
with the breath store and the urge, moved every interval to real seconds,
and added cold shock, breathing as a rate, the ears, the bends, the tread
and the drowning response, the disconnect hold, and air as the rescue for
a breath death; its plan and requirements retired at that sweep.

## River build (2026-10-21)

The tread beat reads the water's **current and roughness** off the medium
(`MediumRead.currentMps`, `roughness`): holding station against a current
costs drag `½ρC_dAv³` every beat, a body not holding is swept (the water
place's `sweep` hook — a river hands it on downstream), rough water takes a
share of the store, and the **feet-first float** (`lie` in moving water,
`isFeetFirst`) halves both. `swimSpeedMps` is a body's own stroke, which the
river's ways read. A water place may close over a body (`headOutAt` false —
a river patch under ice): immersed and not at the surface, the breath model
treats it as under. See [river.md](./river.md).
