# Metabolism

The metabolism subsystem is the body's **intake-and-chemistry driver**:
the consumer the [`Reserve`](./reserve.md) substrate was built for, the
replenishment half of the energy economy [encumbrance](./encumbrance.md)
only drains, and the **first thing in the engine to *drive* vitals
conditions**. It makes `Creature.ingest` real, runs a lazy in-session
reconcile that drives the survival reserves (`satiation` / `hydration`)
and rebuilds `endurance`, spawns/clears [vitals](./vitals.md) conditions
off floored reserves, and (Wave 2) carries a per-toxin burden system with
alcohol/BAC as the worked exemplar.

Everything lives on one mixin — **`MetabolicMixin`**
(`lib/metabolism/Metabolic.ts`). There is **no `MetabolismApi`**: the
clock now-source is the existing `WorldClockApi.getNow()`, the tag-router
is a private method (one caller), and the dials are a module const-object
(`METABOLIC_DEFAULTS`, the `LOAD_BEARING_DEFAULTS` precedent).

## Composition + the lazy time drive

`MetabolicMixin` composes on a `Creature`-shaped host **outer of
`Vitals` / `Reserved` / `Posed`** (it drives all three) and **inner of
`LoadBearing`** (the encumbrance gauge keeps reading the reserve surface
metabolism populates). It overrides **`getReserve` / `getReserves`** to
**reconcile-on-read**: every reserve read — the cockpit poll, the
condition cascade, even `LoadBearing`'s `endurance` read dispatched
through the proxy — first runs `reconcileMetabolism()`, then returns a
fresh value. The overrides read the inner `ReservedMixin` storage
(`reserves` + `reserveFromStored`) directly rather than recursing; a
`_reconciling` reentry guard short-circuits nested reads.

`reconcileMetabolism()` integrates elapsed **in-session game-time** since
the last reconcile (`metabolicClockStamp`), read from
`WorldClockApi.getNow()`. It is a cheap no-op when the world clock isn't
running (most unit tests probe `findByTemplatePath(worldClockRegistry)`
and bail) or when too little time has passed. A long gap is **sub-stepped**
into fixed `STEP_SEC` slices (capped at `MAX_STEPS`, remainder collapsed)
so the coupled flows stay honest; the per-slice order is fixed:

1. **Digestion absorption** — drain each pooled tag toward its target.
2. **Basal drain** — satiation + hydration down, mass-scaled × the Q10
   `thermalMultiplier()` (now lit by the thermal build: `Q10 ^ ((core −
   reference)/10)` off the driven `coreTemperature` — ≈1 for an endotherm
   pinned at setpoint, swings for an ectotherm whose core floats; dials in
   `METABOLIC_DEFAULTS`). See [thermal.md](./thermal.md).
3. **Coupled recovery** — spend both tanks to rebuild endurance (at rest).
4. **Burden clearance** — toxin burdens fall at their clearance rate.

After the slices, `reconcileCascade()` runs once over the final state.

## The digestion buffer

`Creature.ingest(material, amount, phase)` is real (the no-op base seam
moved here; every `Creature` composes the mixin). Intake fills a
**digestion buffer**:

- **Per-tag pools** (`digestionPools: Record<string, number>`) — a bag of
  per-tag quantities, not a list of swallows. Blending old and new intake
  is honest and cheap to persist (decomposed scalars).
- **Two verb-determined sub-volumes** — `solidVolume` (filled by `eat`)
  and `liquidVolume` (filled by `drink`/`sip`), via the `phase` parameter
  on the seam (`'solid'` | `'liquid'`, default liquid). So a body can
  drink when too full to eat with **no `Material` phase flag**.
- **Overeat cap** — intake is accepted up to the remaining sub-capacity
  and the excess refused; `ingest` returns the litres accepted (the bulk
  `result.applied` shape). No separate overeat mechanic.

`ingest` fans the material's tags through **`routeTag`** — a private
static switch over a curated vocabulary (NOT a registry): `water` →
hydration, `carb`/`sugar` → fast satiation, `fat` → slow satiation,
`protein` → the **inert tissue-repair seam** (absorbs out of its pool but
delivers nowhere until vitals healing is driven). Each tag drains toward
its target at **its own rate** — so the vomit window and "eat before the
march" (intake arrives gradually) fall out for free. Toxin tags
(`material.getToxicity()`) add their dose to the pool too (below).

`BulkableApi.ingest` (liquid) and `BulkableApi.ingestSolid` (solid) are
the two Api bridges; `eat`/`vomit` ride `cmd/bulk/` like `drink`/`sip`.
Both accept an optional **`BulkPayload`** (a derived blend's
per-instance face — see [bulk.md](./bulk.md)): when present, its
`nutrients`/`toxicity`/`name` speak for the meal and the slot's generic
base material only supplies physics — so a derived stew nourishes as
the sum of what actually went into the pot (macros in = macros out),
with zero per-dish Material rows.

## Coupled recovery — the keystone

`endurance` does **not** regenerate for free. At rest the body rebuilds it
by **spending `satiation` + `hydration`**, rate-limited and posture-gated:

```
gain = MAX_RECOVERY_PER_MIN × stepMin × postureBase(posture)
       × restQuality(host) × hydrationThrottle × spo2Throttle
```

`postureBase` is lying > sitting > standing (read from `Posed.getPosture()`);
`restQuality` is read from the host whose posture slot the body occupies
(`Slottable.getOccupiedHost()` → `Postured.getRestQuality()`, default 1.0).
**Hydration is the tighter leash** — it costs more per point of endurance
(`RECOVERY_HYDRATION_COST` > `RECOVERY_SATIATION_COST`) and throttles
recovery as it falls. This is *why* recovery lives in metabolism, not
encumbrance: encumbrance spikes are discrete event-drains; metabolism is
the continuous basal + recovery layer. **They layer on the one `endurance`
reserve — the encumbrance build is unchanged.**

`restQuality` is a field on **`PosturedMixin`** (the posture-bearing host),
default 1.0 — a bedroll authors ~1.3×, a four-poster ~2.5×. It is **not on
`SlotSpec`** (the universal slot record stays a pure structural mechanism,
the encumbrance `coupling` precedent); it's a behavior field on the
specialized host, the `Vessel.transmissionFactor` pattern.

## The cascade — metabolism as the first condition-driver

After integration, `reconcileCascade()` reconciles conditions against the
reserve/burden state — the first engine system to drive
[vitals](./vitals.md) conditions:

- For each **floored biological reserve**, ensure its
  `floorEffect`-named `Condition` exists and clear it on recovery
  (hysteresis: spawn at ≤0, clear at ≥ a clear band). `satiation→0` →
  `starvation`, `hydration→0` → `dehydration` (both progress to the death
  seam — `setCauseOfDeath` + `setLifecycleState('dead')` — dehydration
  faster), `endurance→0` → `collapse` (acute, reversible). The
  floor-effect → templatePath map is sourced from
  `TemplatePaths`
  (`metabolismStarvation` / `metabolismDehydration` / `metabolismCollapse`).
- The **death-spiral** is emergent, not special-cased: a collapsed body
  can't run the intake verbs (gated by `requiresConscious`), so satiation
  keeps falling and starvation keeps progressing toward death.

The condition Ideas are authored seeds at
`seeds/lib/metabolism/conditions/`. Metabolism does **not** build a
general progression scheduler — it drives only its own cascade + toxicity
conditions through the reconcile it already has.

### `requiresConscious`

`lib/command/validators/requiresConscious.ts` (sibling of
`requiresAnimate`) fails when the actor isn't conscious
(`getConsciousness() !== 'conscious'`) **or** carries a `collapse`
affliction. It reuses the existing vitals consciousness surface — collapse
layers onto it as a condition the validator reads, no new agency axis.
Composed on the exertion/intake verbs (`eat`, `vomit`, `drink`, `sip`,
`get`, and the locomotion verbs), not on passive perception.

## The in-session clock + presence freeze

The time-drive is **active-play time, not wall-clock** — you get hungry
from *playing*, never from being away. This is the deliberate spot where
the game-design fairness constraint overrides the physics, and the home
for the `WorldClock` time-source cut from encumbrance.

Absence freezes the clock with **zero connection-layer work** — metabolism
reads presence only:

- **Linkdead** — when the host `isHasInteractive` and `isLinkdead()`, the
  reconcile re-stamps and integrates nothing (the body lingers in-world but
  doesn't tick); reconnect resumes as left.
- **Logout** — the avatar leaves the world; its metabolic state rides the
  existing Avatar save/restore. A **far-past guard**
  (`MAX_REASONABLE_GAP_SEC`) drops any gap longer than a plausible
  continuous session, so a relog accrues nothing.

There is **no sleeping player body and no away-recovery** — recovery only
accrues while present and at rest. (Diegetic sleep is a future NPC
behavior over the same posture × `restQuality` rest path.)

## Rates — what the shipped defaults actually mean

Measured 2026-07-31 from `METABOLIC_DEFAULTS`, at the shipped clock scale
of **12×** (1 game day = 2 real hours). These are *played* hours, since the
clock above presence-freezes.

| Reserve | Basal drain | Full → empty | In real play-time |
|---|---|---|---|
| **hydration** | `0.03 %/game-min` | 2.3 game-days | **~4.6 h** |
| **satiation** | `0.02 %/game-min` | 3.5 game-days | **~6.9 h** |

At basal rate a player drinks **about once every two sessions** — deliberately
below the threshold of notice.

### Exertion is the real driver — a ~47× ratio

Basal drain is nearly irrelevant to the felt experience. Coupled recovery
(§ *Coupled recovery*) spends `RECOVERY_HYDRATION_COST = 0.7` hydration per
point of endurance, at up to `MAX_RECOVERY_PER_MIN = 2.0` — so sustained
recovery draws **1.4 %-hydration per game-minute, ~47× basal**, and could drain
a full bar in roughly **six real minutes** at the cap.

> **This ratio is the design, not an accident of tuning.** It means a player who
> talks, trades, and farms never thinks about water, while a player who fights,
> hauls, or treks thinks about it constantly. Consumption is driven by
> **exertion and environment, not by the clock** — which is also the
> physiologically honest model (you lose water sweating, not ticking), and it is
> what the slate's *fun trap* section asks for: gentle at baseline, biting only
> at the margins.

Tuning consequence: **tune the exertion end, not the basal end.** Basal is
already safely below notice; the whole felt curve lives in the recovery draw and
its `RECOVERY_HYDRATION_THROTTLE_PCT = 30` throttle.

### ⚠ `setScale` silently rescales metabolism

Every rate here is **per game-minute**, and game-minutes flow at the world
clock's `scale`. So `WorldClockApi.setScale` is not only "how long is a day" —
it is also **"how fast do you get thirsty, in real time."** Doubling the scale
to 24× halves the real-time hydration budget.

The two are separable decisions that share one number today. If the clock scale
is ever retuned for daylight or seasonal reasons, **the metabolic constants must
be re-derived in the same change**, or survival silently gets harsher or softer
as a side effect. See [time.md § Why 12×](./time.md).

## Wave 2 — meal chemistry

### Inspectable nutrition data + the label

`Material` carries the real, inspectable profile: `nutrientAmounts`
(tag → mg/serving) and `toxicity` reshaped to **`ToxinTag[]`**
(`{type, amount}` — a list of flat objects, default-Hydrator round-trip
like `composition`). The tags drive routing; the amounts are
education-by-reference data. The **render** is `NutritionLabelMixin`
(`lib/metabolism/NutritionLabel.ts`) — an **opt-in consumable affordance**
that appends the label to the host's long description via the
`markupAugmenter` seam (the same mechanism bulk puddles and detail-keys
use). It is **not** a `look`-controller special-case: a label belongs on
things that carry packaging, not on every visible object, and reads only
the item's `Material` (viewer-blind).

### Toxin burdens

A toxin is a **named type** with a **sparse per-body burden scalar**
(`toxinBurdens`, created on first exposure, never widened). A burden is
**not a `Reserve`** — it accumulates unbounded above a threshold
(dose-response), where a Reserve clamps and fires one floor event. The
flow:

- **Ingest** — a `ToxinTag` adds its dose to `digestionPools[type]` like
  any tag (un-absorbed dose is what `vomit` can still dump).
- **Absorption** — the reconcile drains the pool into the burden at the
  toxin's `absorptionRate`: `burden += absorbed × potency / bodyMass`
  (the store-raw alcohol exception below).
- **Clearance** — `burden -= clearanceRate × stepMin`, floored at 0,
  entry deleted at 0.
- **Condition** — each toxin maps to **one banded `Condition`** whose
  severity reads the burden (or BAC) live against the seed-authored
  `bands`; cleared when the level falls below the lowest band. No
  per-band spawn/clear churn.

⭐ **`ToxinTag.labileAtK` — the selective kill.** A dose may declare the
temperature at which a WORKING destroys it (a raw bean's lectin). It rides
the tag the food authors rather than the `Condition` seed, because
lability is a fact about the *substance*: the cook needs to know at what
temperature, while nothing about the body's response changes. Crafting
drops labile doses whose temperature the working actually reached.

⚠ **Three things deliberately author none and honestly survive the pot:**
alcohol; the ptomaine a spoiled input already grew (heat stops the growth,
it does not un-poison what the growth produced — **cooking spoiled food
does not make it safe**); and everything nobody has thought about, because
absence is the safe default. See [spoilage.md](./spoilage.md).

⚠⚠ `Material.setToxicity` rebuilds each entry field by field rather than
spreading it, so a new `ToxinTag` field must be added THERE too or it is
silently dropped at the setter — which is exactly what happened to
`labileAtK` on its first run, with nothing anywhere saying why.

The per-body rate params (`ToxinBehavior`: absorption / clearance /
potency / bands / storeRaw) live on the toxin's **`Condition` seed**
(`Condition.toxinBehavior`), resolved by `<TemplatePathPrefixes.metabolismCondition><type>`
— authored content, not a code table. The toxin types
(`ToxinTag`/`ToxinBand`/`ToxinBehavior` + the Widmark helper) live in
`Metabolic.ts` (types on the owning mixin module, not a standalone file).
v1 ships acute content: alcohol, ptomaine (spoiled food), venom, and lead
(wired, chronic content deferred).

⭐ **Ptomaine is no longer authored per row.** Since the cooking build the
dose is DERIVED from a food's own microbial load at the moment it is
swallowed, on a curve rather than a step, and folded into the ingest
payload at the read — see [spoilage.md](./spoilage.md) § The ingest reach.
The shipped `spoiled-ration` keeps its authored dose as a head start; it
also now grows.

### ⭐⭐ `exposeToPathogens` — where an infection starts

`ingest` is the ONE place food becomes body — every route reaches it (the
discrete arm of `eat`, the dish arm, `drink`, `sip`) — so it is where the
pathogen loads riding a `BulkPayload` become a thing happening inside
somebody. See [spoilage.md](./spoilage.md) for the food side and
[vitals.md](./vitals.md) for what happens next.

Two arms, and only one of them needs anything here:

- **`infect`** — at or above the organism's `infectiousDose`, the
  population is handed to the body as an `AfflictionRecord` carrying a
  live `pathogenLoad`, incubating until `symptomsAt`. A second bad meal
  makes the illness worse rather than starting a second one.
- **`intoxicate`** — nothing to do. The poison was made in the FOOD before
  you picked it up and has already ridden in through the toxin loop above.
  *Killing the population does not unmake it*, which is exactly why
  cooking is not a universal answer.

⚠ A load under the `infectiousDose` does nothing at all. That is not
leniency — it is the difference between "there are some on it" and "you
have eaten enough of them", and it is what makes cooking-and-eating-
promptly a real answer rather than a hope.

⭐ **And the record names the cook.** When the payload carries a `maker`
that is not the eater, an accountability `harm` row is appended (the
trap's producer shape, verbatim). Eating your own risky food is a private
gamble; putting it in front of a paying customer is a choice about
another person, and the ledger is what makes the two different acts.

### `introduceToxin` — the bloodstream seam (past digestion)

`introduceToxin(type, amount)` injects a toxin dose **directly onto the
body burden, bypassing the digestion pool + absorption curve** — the
bloodstream path a poisoned dart or needle uses. Unlike `ingest` (which
fills a pool that the reconcile then drains into the burden at an
`absorptionRate`), the resolved dose lands on the burden immediately; the
banded `Condition` then reads live off the burden (reconcile-on-read),
exactly as an eaten toxin's does. It is *how the dose arrives*, not a new
toxin model. First consumer: a hazard's `delivery.toxin` (a step-dart) —
see [hazard.md](./hazard.md).

### Alcohol / BAC exemplar

The `alcohol` burden is the **store-raw exception**: it stores ethanol
mass (grams) directly, and `getBAC(): Quantity<'g/dL'>` derives BAC via
Widmark — `grams / (bodyMass_kg × r) / 10`, with `r` from
`SexedMixin.getSex()`. A single banded `intoxicated` `Condition` reads
`getBAC()` live against the **`'bac'` tag-scale** (sober → tipsy → drunk →
very-drunk → incapacitated → life-threatening) — drunk and acute
alcohol-poisoning are bands of the **one axis** (the dose makes the
poison). Clearance is **zero-order** (linear BAC fall). The only thing
distinguishing an intoxicant from a pure poison is whether the band map
has a pleasant low rung.

### `vomit` + antidote

`vomit` (`cmd/bulk/vomit.yaml`) flushes the **un-absorbed** pools +
sub-volumes; absorbed state (filled reserves, accumulated burdens) stays.
That asymmetry IS the purge mechanic — inducing it early caps further
absorption (caps drunkenness), late doesn't. The **involuntary** path is
not the verb: the reconcile calls `MetabolicMixin.vomit()` directly on
crossing a toxin's top (acute) band while un-absorbed dose remains, so it
fires on an incapacitated body.

**Antidote = accelerated clearance** — `applyAntidote(toxinType)` crashes
the burden in one shot (far faster than natural clearance), so the banded
condition clears on the next reconcile. It is the minimal consumer of the
vitals `ResolutionSpec` treatment seam (the toxin conditions author
`resolution.by`); no treatment verb is required by this build.

## Units

`lib/quantity.ts` gained `mg` (sub-gram mass; `mg ↔ g` converter — toxin /
nutrient amounts + absorbed-dose math) and `g/dL` (blood-alcohol
concentration), which carries the `'bac'` drunk-ladder scale (registered
in-file after the class). Energy reserves stay `%`; `kcal` is deferred.
See [quantities.md](./quantities.md).

## Inert seams (wired, no driver here)

- **spo2 read** (respiration): `spo2Throttle()` reads `getVitalSign('spo2')`
  and returns 1.0 — lights up when respiration drives spo2.
- **Protein → healing**: routed into an inert pool that drains nowhere.
- **Wired nutrient deficiencies** (scurvy): a deficiency *is* just a
  `Reserve` (the substrate supports it); deferred to a documented seam —
  wire one when a real consumer appears. The education survives in the
  inspectable food data.

## Files

- `lib/metabolism/Metabolic.ts` — the mixin: buffer, reconcile, recovery,
  cascade, toxin burdens, BAC, `vomit`, antidote, dials + toxin types.
- `lib/metabolism/NutritionLabel.ts` — the opt-in consumable label mixin.
- `lib/slot/Postured.ts` — `restQuality` on the posture-bearing host.
- `lib/command/validators/requiresConscious.ts`,
  `lib/command/validators/mustBeEdible.ts`.
- `platform/idea/cmd/bulk/{Eat,Vomit}Controller.ts` + `cmd/bulk/{eat,vomit}.yaml`.
- `seeds/lib/metabolism/conditions/*.yaml` (cascade + toxin conditions),
  `seeds/lib/material/{food,drink}/*.yaml` (toxic + alcoholic materials).
